---
title: "Regeln schreiben"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Regeln schreiben

Dieser Guide zeigt die Rule-DSL so, wie sie praktisch verwendet wird: Bodys aus Atomen aufbauen, über gemeinsame Variablen zwischen Entitäten joinen, Regeln zu größeren Regeln zusammensetzen, mit Negation arbeiten und Regeln im passenden Row-Format ausführen. Das konzeptionelle Material — Projektionen, Ableitungen, Kandidaten — steht unter [Regeln und Ableitungen](/docs/concepts/rules-and-derivations).

## Aufbau einer Regel

Eine `Rule` besteht aus vier strukturell wichtigen Teilen: einer ID, einer Version, einem `select`-Head, der festlegt, was die Regel pro Match zurückgibt, und einem `where`-Body, der die Bedingungen für einen Match enthält. Variablen im Body werden mit dem Context Manager `vars(...)` erzeugt und über den Head projiziert.

```python
from kernel.sdk import Rule, Pred, Not, RuleRef, vars

with vars("p", "n") as (p, n):
    name_lookup = Rule(
        id="q.name",
        version="1.0.0",
        select=[n],
        where=[Person(p), p.name == n],
    )
```

Die Variablen `p` und `n` sind Platzhalter, die im Body gebunden und im Head ausgegeben werden. Der Body sagt: Es gibt eine `Person` `p`, deren `name` den Wert `n` hat. Der Head sagt: Gib für jede passende Bindung `n` zurück. Wird die Regel ausgeführt, entsteht eine Zeile pro Bindung, die den Body erfüllt.

`id` und `version` sind nicht dekorativ. Unter diesen Bezeichnern registriert der Kernel die Regel und verweist in Audit-Records, Evidence-Trees und Referenzen von anderen Regeln auf sie. Man sollte sie wie Funktionsnamen in einer Bibliothek behandeln: stabil wählen und die Version erhöhen, wenn sich die Bedeutung des Bodys ändert.

## Drei Body-Atome

Die meisten Regel-Bodys bestehen aus drei Arten von Atomen, ergänzt durch Entity-Class-Deklarationen, die Variablen auf bestimmte Schemas einschränken.

Das Atom `Pred(predicate, subject, value)` verlangt, dass das Ledger eine aktive Assertion mit dem genannten Prädikat, Subjekt und Wert enthält. Prädikatnamen verwenden die interne kleingeschriebene Form `entity:field`.

```python
Pred("person:tag", p, "vip")  # p has tag "vip"
```

Die typisierte Kurzform derselben Bedingung ist *Feldgleichheit auf Entity-Handles*. Sie ist verfügbar, sobald eine Variable auf eine bestimmte Entitätsklasse eingeschränkt wurde. Die Deklaration `Person(p)` führt `p` als Variable über `Person`-Entitäten ein. Feldzugriffe auf dieser Variable werden dann als Bedingungen im Body verwendet.

```python
Person(p),
p.name == n,
p.locale == "en",
```

`p.name == n` bindet die Variable `n` an den Namen von `p`; `p.locale == "en"` beschränkt die Locale von `p` auf das Literal `"en"`. Beide Formen — `Pred(...)` und `entity.field == ...` — drücken dieselbe Art von Bedingung aus. Die Wahl ist eine Frage der Lesbarkeit, nicht der Leistungsfähigkeit. Die Handle-Form ist vorzuziehen, wenn die Entitätsklasse bereits feststeht, weil sie kürzer ist und lokale Typprüfungen auf den Feldern ermöglicht.

Das Atom `Not([...])` verlangt, dass eine Liste von Clauses nicht gleichzeitig erfüllt ist, unter Negation-as-Failure-Semantik. Es ist die dritte Atomform. Der Abschnitt *Negation in der Praxis* beschreibt die Semantik genauer.

## Über Entitäten hinweg joinen

Eine Regel joint über Entitäten hinweg, indem sie eine Variable zwischen zwei Atomen teilt. Um alle Personen zu finden, die in Deutschland leben, ausgedrückt als Join über die Beziehung `LivesIn` und `Country`, teilt der Body `p` zwischen `Person` und `LivesIn` und `c` zwischen `LivesIn` und `Country`.

```python
with vars("p", "rel", "c") as (p, rel, c):
    germans = Rule(
        id="q.germans",
        version="1.0.0",
        select=[p],
        where=[
            Person(p),
            LivesIn(rel),
            rel.person == p,
            rel.country == c,
            Country(c),
            c.iso_code == "DE",
        ],
    )
```

Die Beziehung `LivesIn` ist eine eigene Entität im Schema; siehe [Ein Schema definieren](/docs/guides/defining-a-schema). Der Join läuft über alle `Person`-Entitäten `p`, die im Feld `person` irgendeiner `LivesIn`-Entität stehen, deren `country` gleich `c` ist, wobei `c` das `Country` mit ISO-Code `DE` ist.

Wenn die Beziehung stattdessen als direktes `entity_ref`-Feld auf der Ausgangsentität modelliert ist — etwa eine `Person` mit einem Inline-Feld `home_country: Country` —, wird der Join kürzer, weil die Referenz bereits Teil der Fakten der Ausgangsentität ist.

```python
with vars("p", "c") as (p, c):
    germans = Rule(
        id="q.germans_direct",
        version="1.0.0",
        select=[p],
        where=[
            Person(p),
            p.home_country == c,
            Country(c),
            c.iso_code == "DE",
        ],
    )
```

In beiden Formen tut die Regel logisch dasselbe: Sie läuft durch den Prädikatgraphen und behält die Bindungen, die alle Clauses des Bodys erfüllen.

## Regeln zusammensetzen

Sobald eine Regel ein nützliches Konzept benennt, kann dieses Konzept über `RuleRef` im Body einer anderen Regel verwendet werden. `RuleRef` fügt den Body der referenzierten Regel in den Body der aufrufenden Regel ein und bindet die Variablen an der Aufrufstelle durch.

```python
with vars("p") as (p,):
    vip_in_good_standing = Rule(
        id="q.vip_ok",
        version="1.0.0",
        select=[p],
        where=[
            Person(p),
            Pred("person:tag", p, "vip"),
            Not([Pred("person:tag", p, "blocked")]),
        ],
    )

with vars("p") as (p,):
    high_priority = Rule(
        id="q.high_priority",
        version="1.0.0",
        select=[p],
        where=[
            RuleRef(vip_in_good_standing)(p),
            Pred("person:tag", p, "active"),
        ],
    )
```

`RuleRef(vip_in_good_standing)(p)` fügt die Bedingung *vip in good standing* als Sub-Clause in den Body der Regel *high priority* ein, wobei `p` als gefundene Person durchgereicht wird. Das ist Komposition, keine Duplikation: Es gibt genau eine Definition von *vip in good standing*, und jede Änderung daran wirkt auf alle Regeln, die sie referenzieren. Gleichzeitig bleiben beide Regelidentitäten in späteren Evidence-Records erhalten, sodass ein Audit-Reader einen *high priority*-Match bis zu der *vip in good standing*-Bedingung zurückverfolgen kann, die ihn legitimiert hat.

Die modulübergreifende Form referenziert eine Regel explizit über ID und Version, ohne dass das Regelobjekt selbst an der Referenzstelle im Scope liegen muss.

```python
RuleRef("q.vip_ok", version="1.0.0")(p)
```

Das ist die idiomatische Form für Referenzen über Modulgrenzen hinweg. Sie erlaubt es, ein Vokabular benannter Regeln aus mehreren Modulen zusammenzusetzen, ohne die entsprechenden Python-Imports erzwingen zu müssen.

## Negation in der Praxis

Das Atom `Not([...])` liest sich natürlich — im Beispiel oben etwa als *not blocked* —, aber seine Semantik ist präzise und sollte verstanden werden. Eine `Not`-Clause ist erfolgreich, wenn das Ledger ihren Body aktuell nicht stützt, und schlägt fehl, wenn das Ledger ihn stützt. Diese Semantik heißt *Negation as Failure*. Im Ledger gibt es keine positive Aussage *not blocked*. Die Regel prüft die Abwesenheit einer Assertion, dass das Subjekt blockiert ist, nicht die Anwesenheit einer Assertion, dass es nicht blockiert ist.

Für den meisten Anwendungscode ist das die richtige Semantik. Praktische Folgen hat der Unterschied aber, wenn Daten über Zeit aus mehreren Quellen eintreffen. Eine Regel, die vor einem Import noch *Alice ist nicht blockiert* ergeben hat, kann danach das Gegenteil ergeben, ohne dass die Regel selbst geändert wurde. Die Schlussfolgerung war korrekt gegen das Ledger, gegen das die Regel lief, aber das Ledger hat sich inzwischen geändert. Der Audit-Trail bewahrt den Ledger-Zustand jedes Regellaufs. Dadurch erscheint die geänderte Schlussfolgerung als Änderung der Inputs und nicht als Inkonsistenz zwischen zwei Läufen. Schlussfolgerungen aus `Not`-Clauses sind aber konstruktionsbedingt defeasible, also widerrufbarer, als Schlussfolgerungen aus positiven Clauses.

Praktisch heißt das: Wenn man zu `Not` greift, sollte man prüfen, ob in der Domäne wirklich die *Abwesenheit* einer Assertion gemeint ist. Manchmal ist das richtig — kein `blocked`-Tag bedeutet legitimerweise nicht blockiert. Manchmal ist die Abwesenheit zu schwach. Dann ist ein expliziter positiver Zustand das bessere Modell, etwa `status == "active"`, den ein Upstream-Akteur affirmativ setzt und den die Regel positiv abfragt, statt sein Komplement zu negieren.

## Eine Regel ausführen

`sdk.run(rule)` gibt eine Liste von Zeilen zurück, deren Form über `row_format` bestimmt wird.

```python
sdk.run(name_lookup, row_format="dict")
# [{"n": "Alice"}, {"n": "Bob"}]

sdk.run(name_lookup, row_format="tuple")
# [("Alice",), ("Bob",)]

sdk.run(germans, row_format="instance")
# [<Person snapshot ...>, <Person snapshot ...>]
```

Das Format `"dict"` ist für Ad-hoc-Arbeit am bequemsten, weil die Keys die Variablennamen aus `select` sind und Zeilen nach Namen gelesen werden können. `"tuple"` lässt die Namen weg und gibt positionale Zeilen zurück. Das passt, wenn die Zeilen an einen nachgelagerten Consumer gehen, der positionale Daten erwartet. `"instance"` ist verfügbar, wenn der Head aus genau einer entitätstypisierten Variable besteht. Dann ist jede Zeile ein Entity-Snapshot in derselben Form, die auch `sdk.get` zurückgeben würde, mit direktem Feldzugriff auf der Zeile.

Ein Standardwert für `row_format` kann beim Öffnen des Stores gesetzt werden und gilt für alle späteren `sdk.run`-Aufrufe, die keinen eigenen Wert angeben.

```python
sdk = SDKStore.from_schema_classes([Person, Country], default_row_format="dict")
```

Ein `row_format=` am einzelnen Aufruf überschreibt den Store-Default.

## Query: die schlankere Form für einmalige Reads

`Query` ist die leichtere Alternative für Inline-Reads, wenn ein Ergebnis gebraucht wird, aber keine benannte, versionierte und wiederverwendbare Regeldefinition.

```python
from kernel.sdk import Query

with vars("p", "loc", "nm") as (p, loc, nm):
    q = Query(
        head=[Person(p), Person.name(locale=loc, name=nm)],
        where=[Person(p), p.locale == loc, p.name == nm],
    )

rows = sdk.run(q)
```

`Query` verwendet dieselbe Body-Sprache wie `Rule`, hat aber keine ID, keine Version und einen reicheren Head, der die Ausgabe direkt formen kann. `Query` ist passend für Reads, die direkt in Anwendungscode eingebettet sind. `Rule` ist passend, wenn das Ergebnis benannt, versioniert, wiederholt ausgeführt oder in weitere Regeln komponiert werden soll.

## Naming und Versionierung

Rule-IDs sind flache Strings. Die Konvention in factpy-Codebasen ist `<kind>.<short_name>`: `q.vip_ok` für Queries, `drv.auto_alias` für Ableitungen, `m.<...>` für Monitoring-Regeln. Der Kernel erzwingt den Prefix nicht, aber bei wachsenden Regellisten zahlt sich die Konvention aus, weil die Art jeder Regel direkt am Namen erkennbar ist.

Die Version einer Regel sollte erhöht werden, wenn sich die *Bedeutung* ihres Bodys ändert — also wenn eine Bindung, die gestern gepasst hätte, heute nicht mehr passt, oder wenn eine Bindung, die gestern nicht gepasst hätte, heute passt. Kosmetische Änderungen, die die Ergebnismenge nicht ändern — umbenannte Variablen, umsortierte Clauses — brauchen keinen Versionssprung. Die Version wandert mit jedem Regellauf in den Audit-Trail. Ein bewusster Versionssprung signalisiert dem Audit-Reader, dass die heutigen Ergebnisse nicht direkt mit denen aus dem letzten Quartal verglichen werden sollten.

Wenn zwei Versionen einer Regel parallel existieren müssen — während einer Migration oder weil mehrere nachgelagerte Consumer unterschiedliche Semantiken brauchen —, sollten sie besser verschiedene IDs bekommen, statt sich nur auf die Versionsstrings zu verlassen. `q.vip_ok` und `q.vip_ok_v2` sind an der Aufrufstelle und in Audit-Records klarer als zwei Versionen derselben ID.

## Nächste Schritte

Der [Guide zum Ausführen von Ableitungen](/docs/guides/running-derivations) behandelt den Candidate-and-Accept-Lebenszyklus für Regeln, deren Head neue Fakten vorschlägt. Der [Guide zur Verwendung von Adaptern](/docs/guides/using-adapters) behandelt das Ausführen derselben Regeln unter PyReason, ProbLog oder Souffle. Das konzeptionelle Bild — was eine Regel ist, warum sie eine Version trägt und warum eine Query eine Projektion ist — steht unter [Regeln und Ableitungen](/docs/concepts/rules-and-derivations).
