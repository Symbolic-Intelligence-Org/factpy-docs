---
title: "Persistenz"
weight: 7
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Persistenz

Standardmäßig lebt das Ledger im Speicher. Es existiert nur so lange wie der Prozess, der es geöffnet hat, und wird beim Beenden des Prozesses verworfen. Für Tests und kurzlebige Berechnungen ist das das passende Verhalten; genau das erzeugt `SDKStore.from_schema_classes([...])`, wenn keine weiteren Argumente übergeben werden. Für alles andere — langlebige Services, Jobs, deren Zustand einen Neustart überleben muss, oder Schemas, die sich über Zeit weiterentwickeln — wird das Ledger in einer Datei persistiert. Dieser Guide erklärt, wie ein persistenter Store geöffnet wird, was der Schema-Digest ist und wovor er schützt, welche Schemaänderungen über Neustarts hinweg sicher sind und welche Migration verlangen, wie mehrere Prozesse ein persistentes Ledger teilen, ohne die Audit-Story zu verlieren, und wie Sidecar-Artefakte und Backups zur Ledger-Datei selbst stehen.

## Einen persistenten Store öffnen

Persistenz wird explizit über das Argument `ledger_path` aktiviert, wenn der Store geöffnet wird.

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes(
    [Person, Country, Order],
    ledger_path="./data/factpy.db",
)
```

Beim ersten Öffnen eines Stores mit einem bestimmten Pfad wird die Datei erstellt und der Digest des Schemas in ihren Metadaten gespeichert. Spätere Schreibvorgänge hängen Einträge in der Reihenfolge an, in der sie passieren. Wird derselbe Pfad erneut geöffnet, berechnet der Kernel den Digest des übergebenen Schemas neu und vergleicht ihn mit dem in der Datei gespeicherten Digest. Stimmen beide überein, wird der Store geöffnet und das Ledger kann gelesen und beschrieben werden. Stimmen sie nicht überein, schlägt das Öffnen mit einem expliziten Fehler fehl, statt den Kernel gegen inkompatible Daten arbeiten zu lassen.

Die Ledger-Datei ist eigenständig. Schema-Digest, Ledger-Einträge und Referenzen auf Sidecar-Artefakte liegen innerhalb des Pfads, der beim Öffnen übergeben wurde. Wird die Datei verschoben, wird der Store als Ganzes verschoben. Es gibt keine zusätzliche Konfiguration, die mitgeführt werden müsste, damit die Datei lesbar bleibt.

## Wovor der Schema-Digest schützt

Der Digest ist ein Content-Hash über das Schema, so wie der Kernel es sieht: jede Entitätsklasse, jedes Identity-Feld mit Primary-Key-Flag und Typbereich, jedes Feld mit Kardinalität und Typbereich sowie die `Meta.version` jeder Entität. Zwei Schemas erzeugen genau dann denselben Digest, wenn ihre strukturell relevanten Teile identisch sind. Das Umbenennen eines Python-Moduls, das Umordnen der Klassen in der Liste an `from_schema_classes` oder das Hinzufügen eines Docstrings ändert den Digest nicht, weil nichts davon verändert, welche Fakten das Schema zulässt oder wie diese Fakten adressiert werden. Das Umbenennen eines Feldes, das Ändern einer Kardinalität oder das Austauschen eines Typbereichs ändert den Digest, weil jede dieser Änderungen die Lesbarkeit von Assertions betrifft, die unter dem alten Schema geschrieben wurden.

Die Prüfung soll keine Änderungen verhindern. Sie soll sicherstellen, dass ein Ledger beim erneuten Öffnen — vielleicht Monate nach dem ersten Schreiben, vielleicht durch einen Prozess, der seine Historie nicht kennt — mit dem übergebenen Schema kohärent gelesen werden kann. Der Fehlerfall, den der Digest ausschließt, ist das stille Fehlinterpretieren alter Daten unter einem veränderten Schema. Dass ein Mismatch laut fehlschlägt, ist genau der Schutz, den der Digest bietet.

## Sichere und unsichere Änderungen

Einige Schemaänderungen erhalten die Lesbarkeit und brauchen weder Digest-Bump noch Migration. Eine neue Entitätsklasse erweitert das Vokabular, ohne bestehende Assertions zu berühren. Ein neues Feld auf einer bestehenden Entität führt ein Prädikat ein, zu dem bestehende Entitäten noch keine Assertions haben; das ist unter den Projektionsregeln ein normaler, wohldefinierter Zustand. Zusätzliche optionale Metadaten — Beschreibungen, Tags oder eine explizite Version, wo zuvor ein impliziter Default galt — beeinflussen den Digest nicht. Ein neues `Identity`-Feld mit `default` oder `default_factory` kann bestehende Identity-Tupel erweitern, ohne sie umzuschreiben, weil der Kernel die neue Koordinate beim Schreiben ergänzen kann, wenn sie nicht angegeben wird.

Andere Änderungen erhalten die Lesbarkeit nicht und bilden eine neue Schemaversion. Das Umbenennen eines Feldes oder einer Entität macht Assertions zum alten Namen unter dem neuen Vokabular unlesbar. Das Ändern der Kardinalität zwischen `single` und `multi` verändert, wie bestehende Assertions in der Projektion reduziert werden. Das Ändern eines Typbereichs — etwa von `string` zu `int` oder von einem Literaltyp zu `entity_ref` — verändert, wie bestehende Werte interpretiert werden. Das Ändern der Identity-Zusammensetzung einer bestehenden Entität invalidiert die Adressen, über die bestehende Assertions gefunden werden. Das Entfernen eines Feldes, auf das bereits geschrieben wurde, lässt verwaiste Assertions im Ledger zurück, für die das neue Schema keinen Platz mehr hat. Jede dieser Änderungen verändert den Digest. Der Kernel verweigert dann das Öffnen des alten Ledgers unter dem neuen Schema, und Migration wird zu einem expliziten Schritt statt zu einer impliziten Neuinterpretation.

Die Grenze ist Lesbarkeit. Eine additive Änderung erweitert das Vokabular, ohne die Bedeutung bestehender Einträge zu verändern, und der Kernel kann mit dem alten Ledger unter dem neuen Schema weiterarbeiten. Eine Änderung, die die Bedeutung bestehender Einträge verändert — indem sie das Prädikat umbenennt, auf das sie sich beziehen, die Projektionsregel ändert, die sie reduziert, oder die Adresse ändert, unter der sie liegen — greift in ihre Bedeutung ein. Der Kernel tut dann nicht so, als wäre alles weiterhin eindeutig.

## Ein Schema migrieren

Wenn eine Änderung die Grenze von lesbar zu nicht lesbar überschreitet, ist Migration der explizite Vorgang, mit dem ein neues Ledger unter dem neuen Schema aufgebaut wird. Der Ablauf besteht im Allgemeinen aus vier Schritten. Die `Meta.version` der betroffenen Entität wird erhöht, sodass die Änderung im Schema selbst sichtbar ist. Das alte Ledger wird unter dem alten Schema in einem separaten Prozess oder In-Memory-Store geöffnet, wo seine Assertions weiterhin zugänglich sind. Diese Assertions werden in ein neues Ledger unter dem neuen Schema replayt, einschließlich aller Transformationen, die die Schemaänderung verlangt. Anschließend wird der Service auf das neue Ledger umgestellt.

Der Kernel stellt bewusst kein In-place-Migrationstool bereit. Eine Migration ist ein kleines Programm, das aus einem Store liest und in einen anderen schreibt. Weil das Ledger append-only ist, können beide Ledgers parallel behalten werden, solange die Audit-Story den Schnitt überbrücken muss. In der Entwicklung, wenn das Schema noch flüssig ist und keine Daten erhalten werden müssen, ist das Löschen der Datei und erneute Öffnen der richtige Weg. Der Digest-Check schützt vorhandene Daten; wo es keine solchen Daten gibt, legt er keine weitere Einschränkung auf.

## Prozessgrenzen überschreiten

Ein persistentes Ledger erlaubt es mehreren Prozessen, die dieselbe Datei verwenden, denselben Zustand zu sehen. Dabei sind zwei Muster zu unterscheiden.

Das erste Muster ist *sequentielle Übergabe*: Prozess A schreibt, Prozess A beendet sich, Prozess B öffnet die Datei und liest. Das ist der häufige Fall — etwa ein Batch-Import, der vollständig durchläuft, gefolgt von einem Service, der Reads aus dem erzeugten Ledger bedient. Die Ledger-Datei ist dabei die ganze Schnittstelle zwischen den Prozessen. Weitere Koordination ist nicht nötig, weil es keine zeitliche Überlappung gibt, in der beide Prozesse die Datei geöffnet halten.

Das zweite Muster ist *gleichzeitiger Zugriff*: Zwei oder mehr Prozesse halten dieselbe Ledger-Datei gleichzeitig offen. Das Locking des Kernels reicht für read-heavy Workloads aus, bei denen es gelegentlich einen einzelnen Writer gibt und der übrige Zugriff aus Reads besteht. Bei Workloads mit mehreren gleichzeitigen Writern ist die passende Architektur meist ein einzelner Writer-Prozess mit mehreren Reader-Prozessen — oder ein gemeinsamer Service, der Writes für mehrere Clients vermittelt — statt mehrerer unabhängiger Prozesse, die parallel anhängen. Das Ledger-Format selbst ist bei gleichzeitigen Reads unproblematisch. Der Konfliktpunkt ist die Metadatenkoordination rund um Writes, und diese ist mit einem einzelnen Writer deutlich leichter zu kontrollieren.

In beiden Mustern bleibt die Audit-Story erhalten: Jede Assertion ist mit Zeitstempel, Quelle und Provenienz versehen, unabhängig davon, welcher Prozess sie erzeugt hat. Dass einige Assertions aus einem anderen Prozess stammen, erscheint im Audit-Record als Metadatum wie jedes andere.

## Sidecar-Artefakte

Fakten mit großen Werten — Dateien, Bilder, Modellgewichte — sollten nicht direkt in der Ledger-Datei liegen. Der Kernel unterstützt dafür das Argument `artifact_store_root`, das ein Verzeichnis angibt, in dem binärer Inhalt separat gespeichert wird.

```python
sdk = SDKStore.from_schema_classes(
    [Person, Document],
    ledger_path="./data/factpy.db",
    artifact_store_root="./data/artifacts",
)
```

Große Werte werden in das Verzeichnis unter `artifact_store_root` geschrieben. Das Ledger hält statt des Werts selbst eine content-adressierte Referenz. Die Audit-Story bleibt dieselbe wie bei jedem anderen Fakt: Die Referenz ist Teil der Assertion, und die Datei bleibt aus dem Paket heraus erreichbar, ohne die Ledger-Datei mit binärem Inhalt aufzublähen.

Wenn `artifact_store_root` nicht angegeben ist, laufen große Werte direkt durch das Ledger. Für Entwicklung und kleine Artefakte ist das akzeptabel, für Produktionssysteme mit größeren Blobs in der Regel nicht. In solchen Fällen sollte die Sidecar-Konfiguration gesetzt werden.

## Backups

Ein Backup eines factpy-Stores besteht aus zwei Teilen: der Ledger-Datei unter `ledger_path` und, falls konfiguriert, dem Verzeichnis unter `artifact_store_root`. Beide sind auf Kernel-Ebene append-only: Das Ledger schreibt frühere Einträge nie um, und der Sidecar-Store schreibt frühere Blobs nie um. Eine Kopie, die angelegt wird, während der Store ruht, ist daher ohne weitere Koordination intern konsistent.

Für ein Backup eines laufenden Stores ist das einfachste korrekte Vorgehen, Writes kurz zu pausieren, die Datei zusammen mit dem Sidecar-Verzeichnis zu kopieren, falls eines verwendet wird, und anschließend fortzufahren. Die entstehende Kopie ist ein vollständiger Snapshot, aus dem der Store an anderer Stelle wieder geöffnet oder replayt werden kann.

Audit-Pakete aus `sdk.export_package` sind ein zusätzliches Archivformat, aber kein Ersatz für Ledger-Backups. Sie erfassen die Records eines oder mehrerer Läufe: was geschrieben, was vorgeschlagen und was akzeptiert wurde. Sie sind das richtige Artefakt für Review und Offline-Prüfung. Als Backup des operativen Zustands reichen sie jedoch nicht aus. Eine Ledger-Datei öffnet man, um mit den Daten weiterzuarbeiten; ein Audit-Paket öffnet man, um zu prüfen, was getan wurde. Beides sollte aufbewahrt werden, aber mit unterschiedlichen Rhythmen und für unterschiedliche Zwecke.

## Nächste Schritte

[Ein Schema definieren](/docs/guides/defining-a-schema) erklärt, welche Änderungen eine neue Schemaversion verlangen. [Einen Lauf auditieren](/docs/guides/auditing-a-run) behandelt, wie Audit-Pakete zu Ledger-Dateien stehen und wann welches Artefakt passend ist. Das append-only-Modell und die Mechanik, mit der Snapshots aus einem persistierten Ledger berechnet werden, stehen unter [Das Ledger](/docs/concepts/the-ledger).
