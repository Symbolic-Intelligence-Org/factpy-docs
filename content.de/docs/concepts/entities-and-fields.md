---
title: "Entitäten und Felder"
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

# Entitäten und Felder

Wenn Fakten die atomaren Aussagen sind, die factpy speichert, dann ist das Schema das Vokabular, in dem diese Aussagen formuliert werden. Es legt fest, über welche Arten von Dingen das System Fakten halten kann, welche Aussagen über sie zulässig sind und wie eine Aussage ihrem Subjekt zugeordnet wird. Ein Schema ist keine Tabellendefinition: Es erzwingt kein Speicherlayout, verlangt nicht, dass ein Feld zu einem bestimmten Zeitpunkt einen Wert hat, und macht keine Aussage darüber, wie die Welt aussehen muss. Es beschränkt die Grammatik des Schreibens und Lesens: die Menge der existierenden Prädikate, ihre Typbereiche und die Regeln, nach denen mehrere Aussagen zum selben Prädikat reduziert werden. Der Rest dieser Seite beschreibt die Bestandteile dieser Grammatik, außerdem wie Schemas über Module hinweg zusammengesetzt werden und wie sie sich von Version zu Version weiterentwickeln.

## Entitäten

Eine Entität deklariert eine Art von Ding, über das das System Fakten halten kann. Die Deklaration erfolgt als Python-Klasse:

```python
from kernel.sdk import Entity, Identity, Field

class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
```

Die Klasse wird nicht instanziiert, um eine Person zu erzeugen. In factpy entsteht nichts durch Klasseninstanziierung, weil der Kernel Fakten in einem Ledger aufzeichnet und die Klasse nur beschreibt, welche Fakten über eine `Person` zulässig sind und wie eine konkrete Person adressiert wird, wenn ein Fakt über sie geschrieben werden soll. Den Kern einer Entität bilden zwei Arten von Deklarationen: Identity-Deklarationen, über die der Kernel eine konkrete Instanz findet, und Field-Deklarationen, die die verfügbaren Prädikate für Aussagen über diese Instanz festlegen.

## Identity

Eine Identity ist eine Koordinate, über die Ledger und SDK die Entität finden, auf die sich ein Fakt bezieht.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
```

Das Flag `primary_key=True` markiert das Feld als Teil der Adresse der Entität. Eine Entität kann mehr als ein solches Feld haben. In diesem Fall muss die Kombination der Werte eindeutig sein:

```python
class User(Entity):
    user_id: str = Identity(primary_key=True)
    locale: str = Identity()
```

Ein `User` wird dann über das Paar `(user_id, locale)` adressiert. Zwei User mit derselben `user_id`, aber unterschiedlicher `locale`, sind verschiedene Entitäten mit getrennten Fakt-Historien. Dieses Muster zusammengesetzter Identitäten ist sinnvoll, wenn eine Entität natürlich abgegrenzt ist — etwa nach Tenant, Region oder Sprache — und diese Abgrenzung wirklich zum Namen der Entität gehört, statt nur ein Feld an ihr zu sein. Wenn zwei solche Instanzen mit unabhängigen Historien nebeneinander existieren sollen, gehört der unterscheidende Wert in die Identity und nicht in ein gewöhnliches Feld.

Eine Identity kann einen Default-Wert oder eine Factory deklarieren, falls beim Schreiben kein Wert angegeben wird:

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
```

`default_factory="uuid4"` weist das SDK an, automatisch einen frischen Identifier zu vergeben, wenn der Aufrufer keinen liefert. `default=...` stellt dagegen einen festen Ersatzwert bereit. Die meisten Domänenentitäten haben einen sinnvollen natürlichen Schlüssel — eine Order-ID, einen ISO-Code, eine Personen-ID — und brauchen keines von beiden. Surrogatschlüssel kommen vor allem bei Entitäten vor, die Beziehungen zwischen anderen Entitäten modellieren.

Identity-Werte sind selbst keine Fakten. Sie werden nicht als Ledger-Einträge gespeichert, unterliegen keiner Projektion oder Reduktion und können nicht zurückgezogen werden, weil sie auf der Adresse der Entität liegen und nicht in ihrer Historie. Alles andere, einschließlich des Landes, in dem eine Person lebt, oder der Adresse eines Firmensitzes, wird als Fakt im Ledger gehalten.

## Felder

Ein Feld ist ein Prädikat, also eine Art von Aussage, die das Schema zulässt. Jede Deklaration legt die Kardinalität des Feldes und seinen Typbereich fest.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
    age: int = Field(cardinality="single")
```

Die Kardinalität `"single"` bedeutet, dass der Snapshot einer Entität zu einem bestimmten Zeitpunkt höchstens einen aktuellen Wert für dieses Feld enthält. Das Ledger akzeptiert und speichert über die Zeit weiterhin beliebig viele Aussagen zu diesem Feld, doch die Projektion in den Snapshot übernimmt nur den neuesten nicht zurückgezogenen Eintrag. Die Kardinalität `"multi"` bedeutet, dass der Snapshot alle aktiven Aussagen zu einer Menge sammelt. Mehrere Aussagen mit demselben Wert fallen in der Projektion zu einem Eintrag zusammen, bleiben im Ledger aber getrennte Einträge.

Kardinalität ist eine Eigenschaft der Projektion, nicht des Ledgers selbst. Das Ledger speichert jede Aussage unabhängig von der Kardinalität. Erst die Reduktionsregel entscheidet beim Berechnen eines Snapshots, ob ein einzelner Wert oder mehrere Werte sichtbar werden. Ein Feld kann außerdem eine optionale `description` enthalten, die in Authoring-Artefakte und Audit-Pakete übernommen wird und das Prädikat direkt dokumentiert.

## Typbereiche

Die Python-Annotation eines Feldes wird auf einen Typbereich des Kernels abgebildet. Verfügbare Bereiche sind `string`, `int`, `bool`, `bytes`, `float64`, `uuid`, `time` für `datetime` und `entity_ref` für Referenzen auf andere Entitäten. Die Annotation `str` wird zu `string`, `int` bleibt `int`, `UUID` wird zu `uuid`, `datetime` zu `time`, und jede Annotation, die selbst eine Subklasse von `Entity` ist, wird zu `entity_ref`.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    home_country: "Country" = Field(cardinality="single")
```

Ein Feld mit dem Typbereich `entity_ref` enthält keinen kopierten Wert aus einer anderen Entität. Es enthält die Adresse, an der diese andere Entität im Ledger liegt. Diese Indirektion erlaubt es Regeln, während der Auswertung von einer Entität zu einer anderen zu laufen. Eine Query wie „finde alle Personen, deren Heimatland Deutschland ist“ funktioniert, weil `home_country` an `Person` eine Referenz auf `Country` hält. Die Regel kann dieser Referenz folgen, den `iso_code` des Landes lesen und nur die Personen behalten, für die dieser Code `"DE"` ist. Der Typbereich eines Feldes wird zusammen mit seinen Aussagen im Ledger aufgezeichnet, bei der Kompilierung von Regeln berücksichtigt, die das Feld verwenden, und fließt in den Schema-Digest ein, den der Kernel beim Öffnen prüft.

## Beziehungen als Entitäten

In den meisten Modellierungssituationen braucht factpy kein eigenes Beziehungskonstrukt. Eine Beziehung zwischen zwei Entitäten ist selbst eine Entität, deren Endpunkte als `entity_ref`-Felder gehalten werden.

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
```

`LivesIn` ist eine gewöhnliche Entität, deren Fakten auf zwei andere Entitäten verweisen. Sie trägt eigene Attribute — hier das Jahr `since` —, sammelt ihre eigene Historie von Aussagen und kann unter derselben DSL Gegenstand von Regeln sein wie jede andere Entität. Die Query „Welche Personen leben seit 2020 in Deutschland?“ wird dadurch zu einem Join über `LivesIn`-Instanzen in derselben Regelsprache, die auch für andere Joins verwendet wird.

Für Systeme, die typisierte Kanten als erstklassiges Kernel-Konstrukt benötigen — also eine explizite `Relationship`-Deklaration mit `from_entity`- und `to_entity`-Typen — stellt der Kernel eine `Relationship`-Basisklasse bereit. Der meiste Anwendungscode braucht sie nicht, weil das Entity-as-Relationship-Muster oben für gewöhnliche Modellierung ausreicht. Nützlich wird die typisierte Kantenform vor allem in Adaptern wie PyReason, wo die Engine selbst auf einem typisierten Graphen statt auf Entitäten und Prädikaten arbeitet.

## Entitätsmetadaten

Eine Entität kann über eine innere `Meta`-Klasse eine kleine Menge Metadaten tragen:

```python
class Person(Entity):
    """A natural person known to the system."""

    class Meta:
        version = "1.0.0"
        description = "Natural person"
        tags = ["domain:identity"]

    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
```

Das Feld `version` ist Teil des Schema-Digests und wird in Regel- und Ableitungsläufe übernommen. So kann ein Audit-Reader unterscheiden, ob Ergebnisse unter einer Version des Entitätsschemas oder unter einer anderen entstanden sind. Das Feld `description` fällt, wenn es fehlt, auf den Docstring der Klasse zurück. `tags` sind freie Labels, die helfen, große Schemas über Teams oder Domänen hinweg zu organisieren. `Meta` akzeptiert nur diese drei Schlüssel. Jeder weitere Schlüssel löst bereits bei der Klassendefinition einen Fehler aus. Diese Enge ist Absicht: Schemadeklarationen sind Teil der Audit-Story, und der Kernel hält die daran beteiligte Oberfläche klein genug, um berechenbar zu bleiben.

## Schemas, Ledger und Digest

Wenn ein persistentes Ledger geöffnet wird, vergleicht der Kernel das übergebene Schema mit einem Digest, der in der Ledger-Datei gespeichert ist. Ändert sich die Kardinalität, der Typbereich oder die Identity-Zusammensetzung eines Feldes so, dass vorhandene Aussagen nicht mehr lesbar wären, schlägt das Öffnen mit einem expliziten Fehler fehl, statt die Daten stillschweigend neu zu interpretieren. Dadurch wird das Schema Teil des Vertrags des Ledgers und nicht bloß ein externer Kommentar dazu: Die Fakten im Ledger sind nur unter dem Schema sinnvoll, das sie geschrieben hat. Eine stille Schemaänderung würde Replay ungültig machen, Audit-Pakete irreführend werden lassen und erlauben, dass abgeleiteter Zustand von den Aussagen abweicht, auf denen er angeblich beruht.

Erweiterungen sind unproblematisch. Ein neues Feld, eine neue Entität oder zusätzliche optionale Metadaten erhalten die Lesbarkeit und beeinflussen den Digest nicht. Das Umbenennen eines Feldes, das Ändern einer Kardinalität oder das Ändern eines Typbereichs erhält die Lesbarkeit nicht, ist eine neue Schemaversion und verlangt, dass ein Ledger aus der alten Version explizit migriert wird, statt es in-place neu zu deuten. Die Mechanik des Digests, der Unterschied zwischen lesbaren und nicht lesbaren Änderungen und Migration als Replay zwischen zwei Ledgers werden unter [Das Ledger](the-ledger.md) und im [Persistenz-Guide](../guides/persistence.md) behandelt.

## Schemas zusammensetzen

Entitätsklassen sind gewöhnliche Python-Klassen in gewöhnlichen Modulen und werden wie jede andere Klasse importiert. Aus Sicht des Kernels ist ein Schema einfach die Liste von Entitätsklassen, die dem SDK beim Öffnen eines Stores übergeben wird.

```python
from kernel.sdk import SDKStore
from myapp.schema.identity import Person, Country
from myapp.schema.commerce import Order, LineItem

sdk = SDKStore.from_schema_classes([Person, Country, Order, LineItem])
```

Es gibt kein zentrales Schema-Registry, keine separate Schema-Datei und keinen Decorator, dessen Ausführung nötig wäre, damit eine Entität erkannt wird. Ein Schema über Module aufzuteilen — nach Domäne, Ownership oder Team — ist derselbe Vorgang wie das Aufteilen jeder anderen Python-Codebasis. Zwei Schemas zu kombinieren ist dasselbe wie Imports aus zwei Packages zusammenzuführen. Architektonisch bedeutet das: Ein Schema kann refaktorisiert, zusammengesetzt, vendored oder extrahiert werden, ohne dass weitere Zeremonie nötig ist. Der Kernel sieht nur die Liste der Klassen, die ihm gegeben wurde, und behandelt sie für die Dauer der Session als Schema.

## Nächste Schritte

[Das Ledger](../the-ledger) beschreibt die Speicher- und Projektionsschicht, in die die Prädikate des Schemas schreiben. [Regeln und Ableitungen](../rules-and-derivations) behandelt die Reasoning-Schicht, in der diese Prädikate verwendet werden. [Audit und Provenienz](../audit-and-provenance) beschreibt, wie das Schema einschließlich seiner Version in Audit-Paketen erfasst wird. Der [Guide zur Schemadefinition](../../guides/defining-a-schema) führt durch reale Schemamuster — zusammengesetzte Identitäten, Surrogatschlüssel und die Weiterentwicklung eines Schemas über Versionen hinweg — für Leser, die dasselbe Material aufgabenorientiert lesen möchten.

