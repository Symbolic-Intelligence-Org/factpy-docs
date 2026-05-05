---
title: "Adapter verwenden"
weight: 8
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Adapter verwenden

Der native Evaluator deckt die Regelform ab, die der meiste Anwendungscode braucht: typisierte Bodys mit Joins, Subregeln und Negation as Failure. Für drei Reasoning-Formen, die der native Evaluator nicht abdeckt — Propagation durch einen Graphen typisierter Beziehungen, probabilistische Inferenz über unsichere Fakten und Datalog-Auswertung in großem Maßstab — delegiert der Kernel an Adapter-Engines. Drei Adapter werden mitgeliefert: PyReason, ProbLog und Souffle. Dieser Guide beschreibt, wann welcher Adapter die richtige Wahl ist und wie die jeweilige Integration grundsätzlich aussieht. Konfigurationsdetails und vollständige APIs stehen auf den Adapter-Referenzseiten.

## Einen Adapter wählen

Die Wahl hängt von der Form des Reasonings ab, nicht von Vorlieben. Der native Evaluator ist der richtige Default, bis es einen konkreten Grund gibt, ihn zu verlassen: Er hat keine zusätzlichen operativen Abhängigkeiten, den schnellsten Iterationszyklus und die kleinste Fehlerfläche bei Modellierungsfehlern.

PyReason passt, wenn das Reasoning die Form von *graphbasierter annotierter Logik* hat. Beliefs sind Intervalle mit unterer und oberer Schranke, sie propagieren entlang typisierter Kanten über diskrete Zeitschritte, und die natürliche Datenform ist ein Graph aus Entitäten und Beziehungen, dessen Zustand sich durch erneute Inferenzläufe verändert. Der Adapter ist die richtige Wahl, wenn Entitäten durch typisierte Beziehungen verbunden sind, Signale mit Confidence-Bändern durch diese Beziehungen propagieren und Zeit Teil des Modells ist.

ProbLog passt, wenn das Reasoning im strengen Sinn *probabilistische Logikprogrammierung* ist. Fakten haben Wahrscheinlichkeiten, Regeln kombinieren sie, und die Antworten sind Wahrscheinlichkeiten über mögliche Ergebnisse. Der Adapter ist die richtige Wahl, wenn Regeln unsichere Inputs auf probabilistisch saubere Weise verarbeiten müssen, einschließlich Marginalisierung über Alternativen, Berechnung gemeinsamer Wahrscheinlichkeiten und Lernen von Wahrscheinlichkeiten aus gelabelten Daten. Er ist nicht die richtige Wahl, wenn Confidences nur heuristische Labels sind und keine kalibrierten Wahrscheinlichkeiten. Dann reicht der native Evaluator mit gewichteten Bodys aus und ist deutlich günstiger.

Souffle passt, wenn das Reasoning plain Datalog ist und das Regelset über das hinausgewachsen ist, was der native Evaluator in akzeptabler Zeit verarbeitet. Die Engine kompiliert Regeln zu nativem Code und wertet sie effizient über sehr große Faktbasen aus. Der Adapter ist die richtige Wahl, wenn Durchsatz auf einer großen Faktbasis zum Bottleneck geworden ist und die Regeln selbst wohlgeformtes Datalog ohne probabilistische Semantik oder Graph-Propagation sind.

Wenn keine dieser Formen das Problem beschreibt, wird kein Adapter gebraucht. Der native Evaluator ist für die große Mehrheit der factpy-Projekte ausreichend.

## ProbLog: Wahrscheinlichkeiten auf Bodys

ProbLog hat die gleichförmigste Integration mit dem SDK. Das Importieren des Adapter-Pakets registriert die Engine im Dispatch von `sdk.evaluate`. Danach kann eine Ableitung mit `mode="problog"` unter ProbLog ausgeführt werden.

```python
import kernel.adapters.problog  # registers the engine evaluator

candidates = sdk.evaluate(some_derivation, mode="problog")
```

Für Ableitungen, die engine-spezifische Konfiguration brauchen — etwa Branch-Wahrscheinlichkeiten oder Inferenzparameter — wird über `engine_ext` eine `ProbLogRuleExt` an die Ableitung gehängt.

```python
from kernel.adapters.problog.rule_ext import ProbLogRuleExt

with vars("r", "name") as (r, name):
    expertise_drv = Derivation(
        id="drv.expertise",
        version="v1",
        where=[Pred("researcher:name", r, name)],
        head=Researcher.expertise(researcher_id=r, expertise=name),
        mode="problog",
        engine_ext=ProbLogRuleExt(branch_probabilities=(0.85,)),
    )
```

Kandidaten aus dem ProbLog-Evaluator tragen Wahrscheinlichkeitsannotationen bis zur Annahme weiter. Beim Akzeptieren werden diese als konventionelle `confidence`-Metadaten auf die entstehenden Ledger-Assertions geschrieben. Die akzeptierten Fakten erscheinen in der Audit-Story wie alle anderen abgeleiteten Fakten; der Unterschied ist, dass der Evidenz-Record den Reasoning-Trace der Engine abbildet und nicht einen flachen Baum.

Die in [Ableitungen ausführen](/docs/guides/running-derivations) eingeführte Form `Body(..., confidence=...)` ist die leichtgewichtige Übergabe zwischen den beiden Evaluatoren: Eine Ableitung mit gewichteten Bodys unter dem nativen Evaluator wird unter ProbLog zu einer Ableitung mit probabilistischen Bodys, ohne dass sich die DSL ändert. Wenn gewichtete Body-Confidences nicht mehr ausreichen — wenn gemeinsame Verteilungen und Marginalisierung nötig werden —, läuft dasselbe `Derivation`-Objekt unter ProbLog, indem nur das `mode`-Argument gewechselt wird.

## PyReason: Graph-Propagation als Session

Die Integration von PyReason ist anders geschnitten, weil die Engine selbst anders arbeitet. Inferenz läuft nicht als einmalige Auswertung gegen ein statisches Ledger. Sie propagiert durch einen Graphen, dessen Zustand sich über diskrete Zeitschritte verändert. Der Adapter stellt diese Form als *Session* bereit: ein zustandsbehaftetes Objekt, in dem der typisierte Graph aufgebaut wird, bevor die Propagation darüber läuft.

```python
from kernel.adapters.pyreason.session import PyReasonSession
from kernel.adapters.pyreason.runner import PyReasonRunConfig, run_pyreason
from kernel.adapters.pyreason.accept import accept_pyreason_session

session = PyReasonSession(schema_ir)

with session.batch() as tx:
    a = tx.entity(Vendor, vendor_id="A")
    b = tx.entity(Vendor, vendor_id="B")
    a.at_risk_signal.set("true", bound=[1.0, 1.0],
        meta={"source": "incident_triage"})
    tx.relationship(VendorDependency, from_entity=a, to_entity=b,
        critical_path="true", bound=[1.0, 1.0])

result = run_pyreason(session, PyReasonRunConfig(...))
accept_pyreason_session(sdk, result, ...)
```

Das Integrationsmuster lautet: typisierten Graphen in einer Session aufbauen, Propagation ausführen und die resultierenden Fakten zurück in den Haupt-Store committen. Der Adapter übernimmt die Übersetzung zwischen factpys Schema und PyReasons annotierter Graphrepräsentation. Entitäten und Beziehungen werden geschrieben wie im SDK, ergänzt um `bound=[lo, hi]` für Confidence-Intervalle auf jedem Fakt und um explizite `Relationship`-Deklarationen für die typisierten Kanten, entlang derer Beliefs propagieren.

PyReason ist die richtige Wahl, wenn das Reasoning selbst Propagation durch Graphzustand ist. Die sessionbasierte API macht genau diese Form ergonomisch. Dieselbe Propagation als flache Ableitung unter dem nativen Evaluator auszudrücken, ist bestenfalls umständlich und wird mit wachsender Propagationstiefe schnell unpraktikabel.

## Souffle: paketbasierte Ausführung

Souffle ist eine kompilierte Datalog-Engine, und die Integration spiegelt das wider. Der Kernel exportiert ein *Paket*, das Regeln, Schema und Input-Fakten beschreibt. Das Souffle-Binary kompiliert und läuft gegen dieses Paket. Die Ergebnisse kommen als neue Fakten zurück.

```python
from kernel.adapters.souffle.package import ExportOptions

# Export an inference package
out = Path("./packages/run-1")
sdk.export_package(out, ExportOptions(package_kind="inference"))

# Run it under Souffle
result = sdk.run_package(out, entrypoints=["my_rule"], engine="souffle")
```

`package_kind="inference"` erzeugt das schlankere Paket für erneute Ausführung statt für Review: Schema, Regeln und Input-Fakten, aber ohne die Audit-Records, die ein Audit-Paket zusätzlich enthält. Das Argument `entrypoints` benennt die Regeln, deren Outputs vom Lauf gewünscht sind. `run_package` ruft das Souffle-Binary gegen das exportierte Paket auf und gibt die abgeleiteten Fakten als Ergebnis zurück.

Souffle ist die richtige Wahl, wenn Regelauswertung unter der nativen Engine zum Bottleneck geworden ist. Der Preis dafür ist ein Paket-Roundtrip pro Lauf, eine Binary-Abhängigkeit von `souffle`, die auf dem Host installiert sein muss, und ein weniger interaktiver Feedbackloop als bei In-Process-Regeln. Der Gewinn ist Durchsatz auf Regelsets und Faktbasen, die der native Evaluator nicht mehr in akzeptabler Zeit verarbeiten würde. Dieselben `Rule`- und `Derivation`-Deklarationen werden verwendet; der Adapter kompiliert sie ohne weiteres Eingreifen des Regelautors in Souffles Oberflächensprache.

## Was über Adapter hinweg gleich bleibt

Drei Eigenschaften gelten einheitlich für den nativen Evaluator und alle drei Adapter. Schema und DSL bleiben unverändert: dieselben `Entity`-Deklarationen, dieselben `Rule`- und `Derivation`-Objekte. Ein Adapter ändert den Evaluator, der sie konsumiert, nicht die Sprache, in der sie geschrieben werden. Die Audit-Story bleibt in ihrer allgemeinen Form unverändert: Jeder Evaluator erzeugt Evidenz, unter der nativen Engine als Baum und unter Engines, deren Reasoning sich nicht auf einen Baum reduzieren lässt, als reicheren `EvidenceGraph`. Audit-Pakete erfassen beide Formen einheitlich. Und der Candidate-and-Acceptance-Lebenszyklus bleibt dort unverändert, wo er anwendbar ist: ProbLog und die kandidatenproduzierenden Teile von Souffle greifen auf `sdk.accept` und `sdk.accept_many` genauso zu wie der native Evaluator. PyReasons sessionbasierter Output läuft über `accept_pyreason_session`, das für diese Engine dieselbe Rolle übernimmt.

Nicht einheitlich sind die Oberflächen für engine-spezifische Konfiguration — Branch-Wahrscheinlichkeiten für ProbLog, Propagation-Step-Counts und Atom-Trace-Flags für PyReason, Dialektversionen und Policy-Modi für Souffle — sowie die Form der Evidenz, die jede Engine erzeugt. Je nach Reasoning-Modell ist sie Baum, Graph oder Timeline. Auch die operativen Abhängigkeiten unterscheiden sich: PyReason und ProbLog werden als optionale Python-Pakete ausgeliefert, während Souffle ein separat installiertes kompiliertes Binary verlangt. Die Adapter-Referenzseiten behandeln diese Oberflächen im Detail.

## Operative Hinweise

PyReason und ProbLog sind optionale Python-Abhängigkeiten und werden nicht standardmäßig mit dem Kernel installiert. Sie werden über die optionalen Extras von factpy-kernel installiert (`pip install factpy-kernel[pyreason]`, `pip install factpy-kernel[problog]`). Die genauen Namen der Extras sollten gegen die Installationsdokumentation des Projekts geprüft werden, da sie sich ändern können. Wenn Adapter-Module importiert werden, ohne dass die zugrunde liegende Engine installiert ist, entsteht ein klarer Fehler statt eines späteren Fehlers während des Laufs.

Souffle benötigt das Binary `souffle` auf dem `PATH` des Hosts. Der Adapter liefert den Compiler nicht mit. Ein fehlendes Binary erscheint als Preflight-Warnung, bevor es als Runtime-Fehler auftritt.

Lokale Regelentwicklung geschieht am besten gegen den nativen Evaluator. Er hat keine zusätzlichen Abhängigkeiten, den schnellsten Iterationszyklus und die kleinste Fehlerfläche bei Modellierungsfehlern. Adapter sind die richtige Wahl, wenn die Reasoning-Form selbst sie verlangt, nicht als Default für gewöhnliche Arbeit.

## Nächste Schritte

[Ableitungen ausführen](/docs/guides/running-derivations) behandelt den Candidate-and-Acceptance-Lebenszyklus, den ProbLog und Souffle mit dem nativen Evaluator teilen. [Einen Lauf auditieren](/docs/guides/auditing-a-run) zeigt, wie von Adaptern erzeugte Evidenz — Graphen und Timelines — in Audit-Paketen erscheint. Die adapterspezifischen Referenzseiten behandeln vollständige Konfiguration, Engine-Extensions und die Datenkonvertierungen, die jeder Adapter durchführt.