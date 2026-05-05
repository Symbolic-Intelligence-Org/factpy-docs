---
title: "Ein Schema definieren"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Ein Schema definieren

Dieser Guide behandelt die praktischen Entscheidungen, die beim Schreiben eines Schemas für eine reale Domäne anfallen: wo das Schema im Projekt liegt, wie Identitäten gewählt werden, wie Kardinalitäten festgelegt werden, wie Beziehungen modelliert werden, wie ein Schema über Module aufgeteilt wird, wie es vor dem Öffnen eines Stores validiert wird und wie es sich von Version zu Version weiterentwickelt. Vorausgesetzt wird das konzeptionelle Material aus [Entitäten und Felder](../../concepts/entities-and-fields). Hier geht es um die Entscheidungen, die konkret anstehen, wenn man zum ersten Mal ein Schema deklariert.

## Wo das Schema liegt

Ein Schema ist eine Sammlung von `Entity`-Subklassen, deklariert in gewöhnlichen Python-Modulen und importiert wie jede andere Klasse. Der Kernel pflegt kein zentrales Registry, braucht keine Manifestdatei und hängt von keinem Decorator ab, der ausgeführt werden müsste, damit eine Entität erkannt wird. Ein typisches Projekt legt sein Schema in einem `schema/`-Package neben dem Anwendungscode ab und gruppiert zusammengehörige Entitäten in Module.

```
myapp/
├── schema/
│   ├── __init__.py
│   ├── identity.py      # Person, Country
│   └── commerce.py      # Order, LineItem
└── app.py
```

Welche Klassen beim Öffnen des Stores an `SDKStore.from_schema_classes(...)` übergeben werden, ist für die Dauer der Session das Schema. Mehr sieht der Kernel nicht.

## Eine Identity-Strategie wählen

Die erste Entscheidung für jede Entität ist, wie sie adressiert wird. Drei Muster decken fast alle Fälle ab, die in der Praxis vorkommen.

Das erste Muster ist der *natürliche Einzelschlüssel*. Er passt immer dann, wenn die Domäne selbst einen sinnvollen eindeutigen Identifier liefert. Ein ISO-Code identifiziert ein Land eindeutig, eine Order-ID identifiziert eine Bestellung eindeutig, und es muss keine weitere Struktur erfunden werden.

```python
class Country(Entity):
    iso_code: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
```

`iso_code` ist die Adresse. `Country(iso_code="DE")` und `Country(iso_code="FR")` sind verschiedene Entitäten mit getrennten Historien.

Das zweite Muster ist die *zusammengesetzte Identity*. Sie passt, wenn eine Entität natürlich scoped ist — etwa nach Tenant, Region oder Sprache — und dieser Scope wirklich zum Namen der Entität gehört, statt nur ein Feld an ihr zu sein. Das Kriterium ist, ob zwei solche Instanzen mit unabhängigen Historien nebeneinander existieren sollen. Wenn ja, gehört der unterscheidende Wert in die Identity und nicht in ein Feld.

```python
class User(Entity):
    user_id: str = Identity(primary_key=True)
    locale: str = Identity()
    name: str = Field(cardinality="single")
```

Der User wird nun über das Paar `(user_id, locale)` adressiert. `User(user_id="u-1", locale="en")` und `User(user_id="u-1", locale="de")` sind verschiedene Entitäten mit getrennten Fakt-Historien. Dieses Muster ist passend, wenn der Scope eine echte Partition der Daten ist — wenn die zwei Locales eines Users wirklich zwei Dinge sind, mit eigenen Namen, eigenen Tags und eigenen Audit-Trails. Es ist unpassend, wenn der Scope nur ein Attribut ist, das sich über die Zeit ändern kann.

Das dritte Muster ist der *Surrogatschlüssel mit Factory*. Er passt, wenn es keinen sinnvollen natürlichen Schlüssel gibt. Der typische Fall ist eine Entität, die nur existiert, um andere Entitäten miteinander zu verbinden: ein Wohnsitz, eine Autorenschaft, eine Review-Zuweisung.

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
```

`default_factory="uuid4"` weist das SDK an, automatisch einen frischen Identifier zu vergeben, wenn beim Schreiben keiner angegeben wird. Die verfügbaren Factories sind in der Referenz dokumentiert; `uuid4` ist die übliche und reicht für fast alle Surrogatschlüssel-Fälle. Die Entscheidung zwischen den drei Mustern lässt sich auf eine Frage reduzieren: Würde das Entfernen des Feldes die Entität in der Domäne uneindeutig machen, gehört das Feld in `Identity`; sonst gehört es in `Field`.

## Kardinalität wählen

Jedes `Field` wird entweder mit `cardinality="single"` oder mit `cardinality="multi"` deklariert. Die Wahl betrifft das, was der Snapshot zeigt, nicht das, was das Ledger speichert. Beide Kardinalitäten erhalten jede Assertion im Ledger. Sie unterscheiden sich nur darin, wie die Projektion mehrere Assertions zu einem sichtbaren Wert reduziert.

Ein Feld mit einfacher Kardinalität passt, wenn eine Entität zu einem Zeitpunkt höchstens einen aktuellen Wert für dieses Feld hat. Ein Name, ein Heimatland, ein Geburtsdatum: jeweils eine Eigenschaft der Entität, bei der eine neue Assertion die vorherige in der Projektion ersetzt.

```python
name: str = Field(cardinality="single")
home_country: Country = Field(cardinality="single")
date_of_birth: datetime = Field(cardinality="single")
```

Ein Feld mit mehrfacher Kardinalität passt, wenn eine Entität eine *Menge* von Werten trägt, die gleichzeitig gelten. Ein Tag, ein Alias, eine bevorzugte Sprache: davon kann eine Entität beliebig viele gleichzeitig haben.

```python
tag: str = Field(cardinality="multi")
alias: str = Field(cardinality="multi")
preferred_language: Language = Field(cardinality="multi")
```

Mehrere `add`-Aufrufe mit demselben Wert fallen im Snapshot zu einem Eintrag zusammen, bleiben im Ledger aber getrennte Einträge. Dadurch lässt sich aus dem Audit-Trail beantworten, wie oft ein bestimmter Tag behauptet wurde, von wem und wann, ohne dass sich die Snapshot-Ansicht verändert. Als Faustregel gilt: Ein Feld mit einer kleinen festen Zahl unterscheidbarer Werte — etwa Status oder primärer Kontakt — ist single-cardinality. Ein Feld mit einer offenen Menge von Werten ist multi-cardinality.

## Beziehungen modellieren

Eine Verbindung zwischen Entitäten kann auf zwei Arten modelliert werden. Die Wahl hängt davon ab, ob die Beziehung selbst etwas aussagen muss oder ob sie nur eine Entität mit einer anderen verbindet.

Die einfachere Form ist eine *direkte Entitätsreferenz*, deklariert als `entity_ref`-Feld auf der Ausgangsentität. Sie passt, wenn die Beziehung rein gerichtet ist und keine eigenen Attribute trägt.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    home_country: Country = Field(cardinality="single")
```

Eine Assertion auf `home_country` zeichnet einen Fakt auf: ein einzelnes Prädikat mit der Adresse des Landes als Wert. Die Beziehung ist querybar, die Projektion ist direkt, und es wird keine weitere Entität gebraucht, um die Verbindung zu tragen.

Die reichere Form ist die *Beziehung als Entität*. Sie passt, wenn die Verbindung eigene Attribute, einen eigenen Lebenszyklus oder eine eigene Historie hat. Dafür wird eine Entität deklariert, deren Felder die Endpunkte der Beziehung und alle weiteren Eigenschaften der Beziehung tragen.

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
    until: int = Field(cardinality="single")
```

Die Beziehung ist nun ein eigenes Subjekt im Schema. Sie sammelt eigene Provenienz, kann eigene Fakten tragen (`since`, `until`) und kann Gegenstand von Regeln sein wie jede andere Entität. Für dieselbe Person können mehrere `LivesIn`-Instanzen existieren, was das passende Modell ist, wenn jemand über die Zeit in mehr als einem Land gelebt hat. Das Entscheidungskriterium ist einfach: Hat die Beziehung selbst etwas auszusagen? Dann verdient sie eine Entität. Wenn nicht, reicht ein `entity_ref`-Feld auf der Ausgangsentität.

## Entitätsmetadaten

Über eine innere `Meta`-Klasse kann eine Entität eine kleine Menge Metadaten in Authoring-Artefakte und Audit-Pakete mitgeben.

```python
class Person(Entity):
    """A natural person known to the system."""

    class Meta:
        version = "1.0.0"
        description = "Natural person"
        tags = ["domain:identity", "owner:platform"]

    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
```

`version` ist Teil des Schema-Digests. Ein bewusster Versionssprung ist dadurch für jeden sichtbar, der ein Ledger öffnet, das unter einer früheren Version geschrieben wurde. `description` fällt, wenn es fehlt, auf den Docstring der Klasse zurück. `tags` sind freie String-Labels, nützlich zum Gruppieren von Entitäten nach Domäne, Ownership oder Compliance-Scope in größeren Schemas. `Meta` akzeptiert nur diese drei Schlüssel. Jeder andere Schlüssel löst bereits bei der Klassendefinition einen Fehler aus. Diese Enge ist Absicht, weil Schemadeklarationen selbst Teil der Audit-Story sind und die daran beteiligte Oberfläche klein gehalten wird.

## Auf Module aufteilen

Schemas wachsen und können entlang jeder Achse aufgeteilt werden, die für das Projekt natürlich ist: nach Domäne, Ownership oder Architekturschicht.

```python
# myapp/schema/identity.py
class Person(Entity):
    ...

class Country(Entity):
    ...

# myapp/schema/commerce.py
from myapp.schema.identity import Person

class Order(Entity):
    order_id: str = Identity(primary_key=True)
    customer: Person = Field(cardinality="single")
    ...
```

Das Schema wird in dem Moment zusammengesetzt, in dem der Store geöffnet wird: Die relevanten Klassen werden importiert und als eine Liste übergeben.

```python
from myapp.schema.identity import Person, Country
from myapp.schema.commerce import Order, LineItem

sdk = SDKStore.from_schema_classes([Person, Country, Order, LineItem])
```

Mehr Zeremonie gibt es nicht. Eine Entität, die über `entity_ref` in einem anderen Modul referenziert wird, muss nur dort importierbar sein, wo sie verwendet wird. Der Kernel interessiert sich nicht dafür, wo eine Klasse deklariert wurde. Bei einem vendored Schema oder einem als Library veröffentlichten Schema werden die Entitätsklassen typischerweise über ein Top-Level-`__init__.py` exportiert, damit Verbraucher sie mit eigenen Klassen kombinieren können.

## Vor dem Öffnen validieren

`schema_preflight_from_classes` führt dieselbe Kompilierung aus wie `from_schema_classes`, gibt aber nur einen strukturierten Report zurück und berührt kein Ledger.

```python
from kernel.sdk import schema_preflight_from_classes

report = schema_preflight_from_classes([Person, Country, Order])
```

Ein fehlender `primary_key`, eine zirkuläre Referenz oder ein Type-Domain-Mismatch erscheinen als strukturierte Einträge im Report, statt erst beim ersten Write als Runtime-Exception aufzutauchen. Den Preflight in Tests oder CI laufen zu lassen, ist der einfachste Weg, Schema-Regressionen früh zu finden, bevor sie ein Ledger erreichen, das auf ein wohlgeformtes Schema angewiesen ist.

## Den Store öffnen

Ein Store kann in zwei Modi geöffnet werden. Der In-Memory-Modus ist für Tests und kurzlebige Berechnungen gedacht. Das Ledger existiert nur für die Lebensdauer des Prozesses und wird beim Beenden verworfen.

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes([Person, Country, Order])
```

Der persistente Modus wird explizit über `ledger_path` aktiviert und ist für langlebige Services oder Zustand gedacht, der einen Restart überleben muss.

```python
sdk = SDKStore.from_schema_classes(
    [Person, Country, Order],
    ledger_path="./data/factpy.db",
)
```

Beim ersten Öffnen eines Stores mit einem bestimmten Pfad wird der Content-Digest des Schemas in der Datei gespeichert. Bei jedem späteren Öffnen wird der Digest des übergebenen Schemas neu berechnet und mit dem gespeicherten verglichen. Wenn sich Kardinalität, Typbereich oder Identity-Zusammensetzung eines Feldes so geändert haben, dass bestehende Assertions nicht mehr lesbar wären, schlägt das Öffnen mit einem expliziten Schema-Mismatch-Fehler fehl, statt die Daten stillschweigend neu zu interpretieren. Diese Prüfung ist der wichtigste Schutz gegen unbeabsichtigten Schema-Drift auf einem persistenten Ledger und sollte ernst genommen werden, wenn sie auslöst.

## Ein Schema weiterentwickeln

Einige Änderungen erhalten die Fähigkeit des Schemas, bestehende Assertions zu lesen, und brauchen weder Versionssprung noch Migration: eine neue Entität hinzufügen, ein neues Feld zu einer bestehenden Entität hinzufügen, optionale Metadaten lockern oder ein neues `Identity`-Feld hinzufügen, das entweder `default` oder `default_factory` hat. Gemeinsam ist diesen Änderungen, dass keine bereits im Ledger liegende Assertion unlesbar wird. Sie erweitern das Vokabular, ohne die Bedeutung des bereits Geschriebenen zu verändern.

Andere Änderungen erhalten die Lesbarkeit nicht und sind eine neue Schemaversion: ein Feld oder eine Entität umbenennen, die Kardinalität eines Feldes zwischen single und multi ändern, den Typbereich eines Feldes ändern, die Identity-Zusammensetzung einer bestehenden Entität ändern oder ein Feld entfernen, auf das bereits geschrieben wurde. Der Kernel verweigert das Öffnen eines Ledgers, das unter einem anderen Schema-Digest geschrieben wurde. Migration ist deshalb eine explizite Operation und keine implizite Neuinterpretation: `Meta.version` der Entität wird erhöht, ein Migrationsprogramm spielt das alte Ledger unter dem neuen Schema in ein neues Ledger ein, und das neue Ledger wird erst geöffnet, nachdem die Migration gelaufen ist. In der ersten Entwicklungswoche mit einem noch flüssigen Schema und ohne Ledger, dessen Kontinuität erhalten werden muss, ist das Löschen der Datei und erneute Öffnen der richtige Weg. Der Digest-Check schützt vorhandene Daten; wo keine solchen Daten existieren, legt er keine weitere Einschränkung auf.

## Nächste Schritte

Der [Guide zum Lesen und Schreiben](../reading-and-writing) behandelt die alltägliche Arbeit mit `set`, `add`, `retract`, Batches und Write-Time-Metadaten. Die [SDK-Referenz](../reference/sdk) listet alle Entry Points mit vollständiger Signatur und Optionen. Den konzeptionellen Hintergrund zu diesen praktischen Entscheidungen bieten [Entitäten und Felder](../../concepts/entities-and-fields) und [Das Ledger](../../concepts/the-ledger).
