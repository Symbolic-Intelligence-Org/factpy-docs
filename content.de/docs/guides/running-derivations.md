---
title: "Ableitungen ausführen"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Ableitungen ausführen

Eine Ableitung ist eine Regel, deren Head neue Fakten vorschlägt, wenn der Body passt, statt Zeilen zurückzugeben. Der Lebenszyklus einer Ableitung besteht aus drei Schritten: Die Auswertung erzeugt Kandidaten aus dem aktuellen Ledger-Zustand, die Prüfung entscheidet, welche Kandidaten zu Fakten werden sollen, und die Annahme schreibt die akzeptierten Kandidaten als neue Ledger-Einträge, wobei ihre Evidenz als Provenienz erhalten bleibt. Dieser Guide zeigt die einzelnen Schritte in der Praxis. Die konzeptionelle Grundlage steht unter [Regeln und Ableitungen](/docs/concepts/rules-and-derivations).

## Eine Ableitung deklarieren

Eine `Derivation` verwendet dieselbe Body-Sprache wie eine `Rule` und trägt ebenfalls ID und Version. Sie unterscheidet sich nur im Head: Eine `Rule` projiziert Variablenbindungen als Zeilen, eine `Derivation` deklariert einen faktförmigen Head, der beschreibt, was für jede passende Bindung vorgeschlagen werden soll.

```python
from kernel.sdk import Derivation, vars

with vars("u", "loc", "nm") as (u, loc, nm):
    auto_alias = Derivation(
        id="drv.auto_alias",
        version="1.0.0",
        where=[User(u), u.locale == loc, u.name == nm],
        head=User.tag(locale=loc, tag=nm),
    )
```

Das `where` ist ein gewöhnlicher Regel-Body. Der `head` ist ein *Feldaufruf* — `User.tag(...)` — und benennt den Fakt, den jeder Match vorschlagen soll. Identity-Koordinaten wie `locale=loc` werden aus den Variablenbindungen des Bodys gelesen; der Feldwert, hier `tag=nm`, ist der Wert, den der Body gebunden hat. Ein Match im Body erzeugt einen Kandidatenfakt im Head.

Eine Ableitung kann pro Match mehrere zusammengehörige Fakten schreiben, indem eine Liste von Head-Feldaufrufen übergeben wird.

```python
with vars("u", "loc", "nm", "tg") as (u, loc, nm, tg):
    multi = Derivation(
        id="drv.multi",
        version="1.0.0",
        where=[User(u), u.locale == loc, u.name == nm, u.tag == tg],
        head=[
            User.name(locale=loc, name=nm),
            User.tag(locale=loc, tag=tg),
        ],
    )
```

Jede Bindung, die den Body erfüllt, erzeugt alle aufgelisteten Head-Fakten als Kandidaten. Das ist die passende Form, wenn mehrere zusammengehörige Fakten eine kohärente Einheit bilden und gemeinsam ausgewertet, geprüft und akzeptiert werden sollen, statt einzeln behandelt zu werden.

## Auswerten

`sdk.evaluate(derivation, mode=...)` führt die Ableitung gegen das aktuelle Ledger aus und gibt eine Liste von Kandidaten zurück.

```python
candidates = sdk.evaluate(auto_alias, mode="native")
print("candidates:", len(candidates))
```

Ein Kandidat ist noch kein Teil des Ledgers. Er ist ein Vorschlag: Prädikat, Subjekt und Wert, die die Ableitung als Assertion vorschlägt, zusammen mit einem Evidenzpaket, das Regel, Version und die Ledger-Einträge nennt, die den Match gestützt haben. Solange der Kandidat nicht akzeptiert ist, hat sich im Ledger nichts geändert. Die Auswertung kann gegen dasselbe Ledger beliebig oft wiederholt werden, um zu prüfen, was vorgeschlagen würde, ohne etwas zu committen.

`mode="native"` wählt den eingebauten Evaluator. Er ist für gewöhnliche logische Ableitungen die richtige Wahl. Für Reasoning-Formen, die der native Evaluator nicht abdeckt — Graph-Propagation, probabilistische Inferenz, großskaliges Datalog — gibt es die Alternativen `"pyreason"`, `"problog"` und `"souffle"`, die unter [Adapter verwenden](/docs/guides/using-adapters) behandelt werden. Dasselbe `Derivation`-Objekt kann unter jeder Engine laufen; geändert wird nur der Evaluator, der es ausführt.

Die Auswertung ist read-only und side-effect-free. Genau deshalb kann sie beliebig oft für Inspektion oder Review vor der Annahme ausgeführt werden, ohne eine spätere Annahme zu beeinflussen.

## Prüfen

Zwischen Auswertung und Annahme liegt der Schritt, in dem entschieden wird, welche Kandidaten zu Fakten werden sollen. Der Kernel schreibt dafür keine bestimmte Form vor, weil sie vom jeweiligen System abhängt. Entscheidend ist nur, dass die Entscheidung zusammen mit den daraus entstehenden Writes aufgezeichnet wird. Erst dadurch wird die Audit-Story später interpretierbar.

In einer automatisierten Pipeline ist die Prüfung meist ein Filter in normalem Code.

```python
to_accept = [c for c in candidates if c.confidence is None or c.confidence >= 0.8]
```

In einem Human-in-the-loop-Workflow werden Kandidaten in einer Oberfläche angezeigt und von einem Reviewer einzeln oder im Batch akzeptiert. Bei einer Ableitung, der per Policy vollständig vertraut wird und die keine Prüfung pro Kandidat braucht, reduziert sich der Filter auf die Identität: `to_accept = candidates`.

Die konkrete Form der Entscheidung ist weniger wichtig als die Tatsache, dass sie getroffen und aufgezeichnet wird. Die Annahme schreibt zu jedem Ledger-Write einen Eintrag ins Decision Log, einschließlich Reviewer oder Policy, die den Kandidaten freigegeben hat. Die zentrale Eigenschaft lautet: Jeder Fakt im Ledger wurde entweder direkt geschrieben oder durch eine identifizierbare Entscheidung akzeptiert. Genau das macht den Audit-Trail verständlich, wenn ein Lauf später von Dritten geprüft wird. Der Decision-Log-Eintrag ist die Verbindung zwischen Kandidat und den Writes, die er erzeugt hat.

## Akzeptieren

`sdk.accept(candidate, approved_by=..., note=..., dry_run=...)` schreibt einen einzelnen Kandidaten ins Ledger.

```python
result = sdk.accept(candidate, approved_by="reviewer-44", note="bulk import")
```

Die Argumente `approved_by` und `note` fließen ins Decision Log. Die Evidenz des Kandidaten — Regel-ID, Regelversion und stützende Ledger-Einträge — bleibt als Provenienz an der neuen Assertion erhalten. Acceptance ist idempotent: Wird derselbe Kandidat ein zweites Mal akzeptiert, entsteht kein weiterer Write. Der Duplikatversuch wird im Decision Log festgehalten.

Für einen Batch von Kandidaten ist `accept_many` effizienter und bietet einen Parameter zur Steuerung der Atomarität.

```python
result = sdk.accept_many(to_accept, mode="atomic")
```

`mode="atomic"` schreibt alles oder nichts: Entweder werden alle Annahmen im Batch erfolgreich geschrieben und gemeinsam sichtbar, oder keine. `mode="best_effort"` schreibt, was geschrieben werden kann, und gibt ein Ergebnis zurück, das Fehlschläge separat ausweist. Der atomare Modus passt, wenn die Kandidaten eine kohärente Einheit bilden — etwa wenn eine Ableitung zu einem User gemeinsam Name, Tier und Flag vorschlägt und alle drei nur zusammen sinnvoll sind. Best-Effort passt für unabhängige Kandidaten, bei denen teilweiser Fortschritt besser ist als keiner.

Sowohl `accept` als auch `accept_many` akzeptieren `dry_run=True`. Damit werden die Writes nur vorgerechnet, ohne sie zu committen. Das Ergebnis hat dieselbe Form wie beim echten Schreiben und eignet sich für einen Preflight-Check oder für ein Interface, das einem Reviewer zeigt, was eine Annahme tun würde, bevor sie ausgeführt wird.

## Bodys und Confidence

Eine Ableitung kann mehrere Wege zur selben Schlussfolgerung haben, mit unterschiedlichem Vertrauen in jeden Weg. Der Wrapper `Body(...)` macht diese Pfade explizit und versieht sie mit einem Confidence-Wert.

```python
from kernel.sdk import Body, Pred

with vars("u", "loc", "tg") as (u, loc, tg):
    vip_inference = Derivation(
        id="drv.vip",
        version="1.0.0",
        where=[
            Body([User(u), u.locale == loc, Pred("profile:vip", u, True)],
                 confidence=0.95),
            Body([User(u), u.locale == loc, Pred("model:high_value", u, tg)],
                 confidence=0.6),
        ],
        head=User.tag(locale=loc, tag="vip"),
    )
```

Jeder `Body` ist ein alternativer Match-Pfad. Die Bodys werden per Oder verbunden: Ein Kandidat entsteht, wenn irgendeiner der Bodys für eine Bindung erfüllt ist. Kandidaten tragen die Confidence des Bodys, der sie erzeugt hat, als opake Metadaten. Ein Review-Schritt kann darauf reagieren, etwa Kandidaten mit hoher Confidence automatisch akzeptieren und Kandidaten mit niedriger Confidence an einen menschlichen Reviewer weiterleiten.

Wenn Confidence im strengen probabilistischen Sinn gemeint ist und die Engine Confidences korrekt unter gemeinsamen Verteilungen und Marginalisierung kombinieren soll, übergibt factpy an ProbLog über die optionale Adapter-Schicht. Die `Body`-und-Confidence-Form ist der Übergabepunkt des Kernels: Unter dem nativen Evaluator reisen Confidence-Werte als opake Zahlen mit; unter ProbLog werden sie als Wahrscheinlichkeitsannotationen interpretiert und kombiniert. Dasselbe `Derivation`-Objekt kann unter beiden Evaluatoren laufen. Was sich ändert, ist die Semantik des Confidence-Werts.

## Evidenz nachträglich lesen

Nachdem ein Kandidat akzeptiert wurde, trägt die daraus entstandene Assertion im Ledger einen Verweis auf die Regel und Version, die sie erzeugt hat, sowie auf die stützenden Ledger-Einträge. Die Lineage kann direkt aus den Metadaten der Assertion gelesen werden. Für eine strukturierte Sicht über viele abgeleitete Fakten hinweg kann ein Audit-Paket des Laufs mit `kernel.audit` geöffnet und die Evidenzbäume können durchlaufen werden.

```python
from kernel.audit import load_audit_package

pkg = load_audit_package("./audits/run-2024-04-01")
for tree_id, tree in pkg.provenance_trees.items():
    # each tree shows a derived fact and its supporting entries
    ...
```

Siehe [Audit und Provenienz](/docs/concepts/audit-and-provenance) für das konzeptionelle Material dazu, was während eines Laufs aufgezeichnet wird, und den [Guide zum Auditieren eines Laufs](/docs/guides/auditing-a-run) für die Praxis des Erzeugens und Lesens eines Audit-Pakets.

## Muster und Fallstricke

Einige Praktiken tauchen in factpy-Codebasen regelmäßig auf und sollten ausdrücklich benannt werden.

Eine Ableitung erneut auszuführen ist sicher. Die Auswertung erzeugt Kandidaten aus dem aktuellen Ledger, ohne es zu verändern. Die Annahme ist idempotent: Ein bereits akzeptierter Kandidat erzeugt keinen zweiten Write, und der Duplikatversuch wird im Decision Log festgehalten. Es gibt kein operatives Risiko dabei, dieselbe Ableitung mehrfach auszuwerten, sei es zur Inspektion der Vorschläge unter verschiedenen Ledger-Zuständen oder zur Wiederherstellung eines Kandidatensets nach einem unterbrochenen Review.

Bodys in einer Multi-Body-Ableitung sind oder-verknüpft, nicht und-verknüpft. Ein Kandidat entsteht, wenn irgendeiner der Bodys passt, nicht erst, wenn alle passen. Für eine und-verknüpfte Bedingung verwendet man einen einzelnen Body mit allen Clauses. Die Multi-Body-Form existiert, um alternative Wege zur selben Schlussfolgerung mit potenziell unterschiedlichen Confidences auszudrücken. Diese beiden Formen zu verwechseln ist der häufigste Modellierungsfehler beim ersten Einsatz von `Body(...)`.

Versionierung ist bei Ableitungen genauso wichtig wie bei Regeln, und aus demselben Grund. Wenn sich die Bedeutung des Bodys einer Ableitung ändert, sollte die Version erhöht werden. Die neue Version erzeugt neue Kandidaten. Die unter der alten Version akzeptierten Fakten bleiben mit ihrer ursprünglichen Provenienz im Ledger und sind weiter auf die Regelversion zurückführbar, die sie erzeugt hat. Der Audit-Reader sieht beide Versionen als unterschiedliche Ursprünge unterschiedlicher Fakten und wird nicht dazu verleitet, Ergebnisse unter einer Version direkt mit Ergebnissen unter einer anderen zu vergleichen.

Auto-Acceptance ist als bewusste Policy angemessen, aber nicht als Default. Der Candidate-and-Acceptance-Lebenszyklus existiert, damit hinter jedem Fakt im Ledger eine identifizierbare Entscheidung steht. Auch wenn diese Entscheidung automatisiert ist, bleibt der Decision-Log-Eintrag die Verbindung, die die Audit-Story kohärent macht. Eine Pipeline, die `sdk.evaluate` und `sdk.accept_many` ohne Filter aufruft, ist dann sauber modelliert, wenn die ausgesprochene Entscheidung lautet: *Alle Kandidaten dieser vertrauenswürdigen Ableitung werden per Policy akzeptiert.* Diese Policy muss aber eine bewusste Wahl sein und nicht bloß die Folge eines weggelassenen Filters.

## Nächste Schritte

Der [Guide zum Auditieren eines Laufs](/docs/guides/auditing-a-run) behandelt den Export des Pakets, das ein Ableitungslauf erzeugt, und das Wiedereinlesen mit `kernel.audit`. Der [Guide zur Verwendung von Adaptern](/docs/guides/using-adapters) behandelt das Ausführen derselben Ableitungen unter PyReason, ProbLog oder Souffle. Das konzeptionelle Modell — was ein Kandidat ist, warum Evidenz mitläuft und warum Annahme ein eigener Schritt ist — steht unter [Regeln und Ableitungen](/docs/concepts/rules-and-derivations).
