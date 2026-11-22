---
layout: distill
title: "Reproduzierbar heißt nicht verifiziert: Lehren aus dem PCA-Detektor von PyOD"
description: "Gepinnte Versionen machen Ergebnisse reproduzierbar, verifizieren aber nicht den Code. Wird derselbe Code erneut ausgeführt, liefert er dieselben Zahlen – unabhängig davon, ob er die zitierte Methode implementiert. Wir veranschaulichen dies am PCA-Detektor von PyOD, dessen Anomalie-Score von der in seiner Dokumentation referenzierten Methode abweicht. Die Abweichung ist seit Jahren öffentlich gemeldet, dennoch wurde der Detektor zur Baseline in Benchmarks und Papern. Angelehnt an die Erfassung von Sicherheitslücken skizzieren wir Correctness Advisories, die bekannte Abweichungen mit den betroffenen Ergebnissen verknüpfen."
date: 2027-04-30
future: true
htmlwidgets: true
hidden: true

# Mermaid diagrams
mermaid:
  enabled: true
  zoomable: true

# Anonymize when submitting
authors:
  - name: Anonymous

# must be the exact same name as your blogpost
bibliography: 2027-04-30-pca-pyod.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - please use this format rather than manually creating a markdown table of contents.
toc:
  - name: Einleitung
  - name: "Fallstudie: PCA in PyOD"
    subsections:
      - name: Was die zitierte Methode berechnet
      - name: Was PyOD berechnet
      - name: "Problem 1: Der Score liegt nahe an der Norm"
      - name: "Problem 2: Das Vorzeichen"
      - name: Ein kurzes Beispiel
      - name: Die Korrektur
  - name: Wie weit es sich verbreitet hat
  - name: Warum es überlebt hat
  - name: Was fehlt
  - name: Ein Vorschlag für eine Community-weite Lösung
    subsections:
      - name: Einschränkungen
  - name: Fazit
    subsections:
      - name: Ausblick
      - name: Abschließende Gedanken
  - name: Anhang A
  - name: Anhang B
  - name: Anhang C
  - name: Anhang D

_styles: >
  mjx-container[display="true"] {
    overflow-x: auto;
    overflow-y: hidden;
  }
  .language-diff .highlight .gd { background-color: rgba(248, 81, 73, 0.18); color: inherit; }
  .language-diff .highlight .gi { background-color: rgba(46, 160, 67, 0.18); color: inherit; }

---

## Einleitung

Jeden Tag bauen wir auf Code auf, den wir nie gelesen haben. Wenn wir eine Bibliothek auswählen, überfliegen wir ihre Dokumentation, prüfen, ob andere sie nutzen, und machen weiter. Nur wenige Forschende lesen den Quellcode im Detail. Generative KI vergrößert diese Lücke, da immer mehr Code und Text entstehen, ohne dass jemand jedes Detail durcharbeitet. Shah <d-cite key="researchAskingAuthorsTheir2026"></d-cite> berichtet von Einreichungen, die ihre Autor:innen nicht erklären konnten, und äußert die Sorge, dass solche Paper KI-generiert oder stark KI-unterstützt sein könnten. Wenn Autor:innen ihre Paper nicht vollständig verstehen, ist es noch unwahrscheinlicher, dass die zugrunde liegenden Bibliotheken auf Fehler geprüft werden.

Reproduzierbarkeit schützt davor nicht. Gepinnte Versionen machen Ergebnisse reproduzierbar. Derselbe Code liefert dieselben Ergebnisse <d-cite key="nasem2019reproducibility"></d-cite>. Ob der Code die Methode implementiert, der er zugeschrieben wird, ist eine separate Frage, die als Codeverifikation bezeichnet wird <d-cite key="roy2005verification"></d-cite>. Veröffentlichte Paper sind unveränderlich, der zugrunde liegende Code kann sich jedoch ändern. Derzeit gibt es keinen Mechanismus, der Autor:innen, Gutachter:innen und Leser:innen darüber informiert, dass der Code nicht so implementiert ist, wie in den Referenzen beschrieben.

Auf einen solchen Fall sind wir gestoßen, als wir uns mit Anomalieerkennung auf Basis der Hauptkomponentenanalyse (Principal Component Analysis, PCA) in [PyOD](https://github.com/yzhao062/pyod) <d-cite key="zhao2019pyod"></d-cite> beschäftigt haben. PyOD dokumentiert seinen PCA-Detektor mit einem Verweis auf den Principal Component Classifier (PCC) von Shyu et al. <d-cite key="shyu2003"></d-cite> und die von Aggarwal <d-cite key="aggarwal2017"></d-cite> beschriebenen PCA-basierten Methoden. Die in PyOD implementierte Berechnung des Anomalie-Scores weicht jedoch von der in diesen Referenzen beschriebenen ab. Da PyOD sich selbst als „meistgenutzte Python-Bibliothek für Anomalieerkennung“ <d-cite key="yzhao062"></d-cite> bezeichnet, hat sich der abweichende Score in die Forschung, etwa in den Benchmark ADBench <d-cite key="NEURIPS2022_cf93972b"></d-cite>, und in Produkte wie PostHog <d-cite key="posthog"></d-cite> verbreitet.

Zunächst nehmen wir den Fall auseinander. Wir rekapitulieren, was die zitierte Methode berechnet, zeigen, wo PyOD davon abweicht, und schlagen eine Korrektur vor. Anschließend fragen wir, wie weit sich der abweichende Score verbreitet hat, warum er überlebt hat, obwohl er seit 2019 öffentlich gemeldet ist, und was fehlt, um solche Probleme sichtbar zu machen. Wir schließen mit einem Vorschlag für Correctness Advisories (Korrektheitshinweise), die auf bestehender Infrastruktur aufbauen.

## Fallstudie: PCA in PyOD

### Was die zitierte Methode berechnet


Die Hauptkomponentenanalyse (PCA) ist ein klassisches Verfahren zur Dimensionsreduktion <d-cite key="Jolliffe2002Principal"></d-cite>. Sie bestimmt orthogonale Richtungen, die Hauptkomponenten, geordnet danach, wie viel der Varianz der Daten sie erklären. Die Idee hinter PCA-basierter Anomalieerkennung ist, dass Beobachtungen, die außerhalb des von den Major-Hauptkomponenten aufgespannten Unterraums liegen, als Anomalien klassifiziert werden. Während bei der Dimensionsreduktion typischerweise nur die Major-Komponenten behalten werden, um die dominante Varianz abzubilden, kann PCA-basierte Anomalieerkennung zusätzlich die Minor-Komponenten nutzen. Die Major-Komponenten beschreiben die dominante Variation normaler Daten. Die Minor-Komponenten haben minimale Varianz und weisen auf nahezu lineare Abhängigkeiten zwischen den Merkmalen hin; daher reagieren sie empfindlich auf Beobachtungen, die die Korrelationsstruktur verletzen <d-cite key="shyu2003"></d-cite>.

Shyu et al. <d-cite key="shyu2003"></d-cite> beschreiben einen Principal Component Classifier zur Anomalieerkennung. Wir fassen die Teile zusammen, die für den Anomalie-Score benötigt werden. Der vollständige Klassifikator ist in [Anhang A](#appendix-a) beschrieben. Jedes Merkmal wird zunächst mit seinem Stichprobenmittelwert und seiner Stichprobenvarianz standardisiert. Dadurch wird der Schwerpunkt der Daten in den Ursprung verschoben, und jedes Merkmal erhält die Varianz eins. Eine standardisierte Beobachtung bezeichnen wir mit $$\mathbf{z} = (z_1, \dots, z_p)^\top$$.

**Hauptkomponenten**\
Eigenwerte und Eigenvektoren werden per Eigenwertzerlegung aus der $$p \times p$$-Korrelationsmatrix $$\mathbf{R}$$ berechnet. Dies ist die Kovarianzmatrix der standardisierten Werte $$\mathbf{z}$$. Sie liefert $$p$$ Paare aus Eigenwert und normiertem Eigenvektor $$(\lambda_i, \mathbf{e}_i)$$, so geordnet, dass $$\lambda_1 \geq \lambda_2 \geq \dots \geq \lambda_p$$.

Die zentrale Größe ist die $$i$$-te Hauptkomponente $$y_i$$. Sie wird berechnet, indem jede standardisierte Beobachtung $$\mathbf{z}$$ auf den $$i$$-ten Eigenvektor $$\mathbf{e}_i$$ projiziert wird:

<p>
$$y_i = \mathbf{e}_i^\top \mathbf{z} = e_{i1} z_1 + e_{i2} z_2 + \dots + e_{ip} z_p $$
</p>

Die Hauptkomponente $$y_i$$ hat auf den Trainingsdaten die Varianz $$\lambda_i$$, daher stellt $$y_i^2 / \lambda_i$$ den quadrierten, varianznormierten Abstand vom Ursprung entlang der $$i$$-ten Hauptachse dar. Die Summe von $$y_i^2/\lambda_i$$ über alle $$p$$ Komponenten ergibt die quadrierte Mahalanobis-Distanz <d-cite key="shyu2003"></d-cite>, die Aggarwal <d-cite key="aggarwal2017"></d-cite> als *Soft PCA* bezeichnet.

**Major- und Minor-Komponenten**\
Anstatt über alle Komponenten zu summieren, werten Shyu et al. die ersten $$q$$ (Major-) und die letzten $$r$$ (Minor-)Komponenten mit zwei separaten Statistiken aus, $$T_{\text{major}}$$ und $$T_{\text{minor}}$$, jeweils eine Summe von $$y_i^2/\lambda_i$$ über die zugehörigen Komponenten ([Anhang A](#appendix-a)). In der Minor-Statistik verstärkt das Gewicht $$1/\lambda_i$$ kleine Abweichungen, die die normale Korrelationsstruktur verletzen. Eine Beobachtung wird als anomal markiert, wenn eine der beiden Statistiken ihren Schwellenwert überschreitet.

<a id="what-pyod-computes"></a>
### Was PyOD berechnet

Der PCA-Detektor von PyOD standardisiert die Daten, fittet die [PCA](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.PCA.html) von scikit-learn <d-cite key="scikitlearn"></d-cite> und bewertet Beobachtungen mit den Explained Variance Ratios (Anteilen der erklärten Varianz) als Gewichten. Diese Pipeline weicht in mehreren Schritten vom Principal Component Classifier von Shyu et al. ab; wir listen sie in [Anhang B](#pyods_pca) auf. Die meisten dieser Abweichungen sind Vereinfachungen. Der Score ist jedoch eine andere Funktion. Wir analysieren die PCA von PyOD in Version `3.6.8` <d-cite key="pyodPCA368"></d-cite>.

Mit den Standardparametern berechnet PyOD den Score wie folgt <d-cite key="pyodPCA368"></d-cite>:

```python
# fit: aus den standardisierten Trainingsdaten gelernt
self.selected_components_ = self.detector_.components_
self.selected_w_components_ = self.detector_.explained_variance_ratio_
# decision_function: X = standardisierte Beobachtungen
np.sum(cdist(X, self.selected_components_) / self.selected_w_components_, axis=1)
```

Die Zeilen von `components_` sind die normierten Eigenvektoren $$s_i\mathbf{e}_i$$, mit einem Vorzeichen $$s_i \in \{-1,+1\}$$, das scikit-learn per Konvention festlegt ([Problem 2](#problem-2)). Der Aufruf von `cdist` misst den euklidischen Abstand jeder Beobachtung zu jedem Eigenvektor, behandelt als *Punkt* an der Spitze des Einheitsvektors. Das ist weder der Abstand zur Hauptachse noch die Projektion $$y_i$$. Jeder Abstand wird anschließend durch das Explained Variance Ratio $$\lambda_i / \sum_j \lambda_j = \lambda_i / p$$ geteilt, und die Ergebnisse werden über alle Komponenten summiert:

<p>
$$
\text{score}_{\text{PyOD}}(\mathbf{z}) = p \sum_{i=1}^{p} \frac{\lVert \mathbf{z} - s_i\, \mathbf{e}_i \rVert}{\lambda_i},
\qquad s_i \in \{-1, +1\}.
$$
</p>

Er weicht in zwei grundlegenden Punkten von den zitierten Referenzen ab.

<a id="problem-1"></a>
### Problem 1: Der Score liegt nahe an der Norm

Jeder Term des Scores ist der Abstand von $$\mathbf{z}$$ zu einer Eigenvektorspitze $$s_i \mathbf{e}_i$$, multipliziert mit dem Gewicht $$p/\lambda_i$$. Alle Spitzen liegen im Abstand 1 vom Zentrum, da sie Einheitsvektoren sind. Der Abstand von $$\mathbf{z}$$ zu einer Spitze ist daher fast derselbe wie der Abstand von $$\mathbf{z}$$ zum Zentrum: Nach der Dreiecksungleichung beträgt er mindestens $$\lVert \mathbf{z}\rVert - 1$$ und höchstens $$\lVert \mathbf{z}\rVert + 1$$. Jeder Term ist also ungefähr $$\lVert \mathbf{z}\rVert$$ mal sein Gewicht, und der gesamte Score ist ungefähr

<p>
$$
\text{score}_{\text{PyOD}}(\mathbf{z}) \approx W \cdot \lVert \mathbf{z}\rVert, \qquad W = p\sum_{i=1}^{p} \frac{1}{\lambda_i}.
$$
</p>

$$W$$ hängt nur von den Eigenwerten der Trainingsdaten ab und ist für jede Beobachtung gleich. Der Score ist daher im Wesentlichen der Abstand zum Zentrum, multipliziert mit einer Konstanten. Die Richtung, in die eine Beobachtung abweicht, kann ihren Score um höchstens $$\pm W$$ verschieben, also um so viel wie eine Einheit der Norm.

**Wann die Eigenwerte eingehen**\
Der entscheidende Unterschied liegt darin, wann die Eigenwerte in den Score eingehen. Shyu et al. skalieren zuerst jede Richtung einzeln und messen dann den Abstand, d. h. jede Hauptkomponente $$y_i$$ wird durch $$\sqrt{\lambda_i}$$ geteilt. Ein kleiner Schritt entlang einer Minor-Komponente zählt damit so viel wie ein großer Schritt entlang einer Major-Komponente. PyOD geht umgekehrt vor. Es berechnet zuerst den euklidischen Abstand zur Spitze $$s_i \mathbf{e}_i$$, in dem alle Richtungen gleich zählen, und teilt erst danach durch $$\lambda_i$$. Diese Division skaliert den gesamten Abstand um denselben Faktor, unabhängig davon, in welche Richtung die Beobachtung abweicht. Die Information darüber, welche Richtungen normal sind, geht also verloren, bevor der Eigenwert überhaupt angewendet wird.

Die Norm ignoriert, wie die Merkmale korreliert sind – und genau diese Struktur soll die PCA erfassen. Anomalien, die die Korrelation verletzen, aber nicht ungewöhnlich weit vom Zentrum entfernt liegen, sind daher schwer zu erkennen. Der Effekt wächst mit der Dimension: Für standardisierte Daten gilt $$\mathbb{E}\lVert \mathbf{z}\rVert^2 = \operatorname{tr}(\mathbf{R}) = p$$, im Mittel also $$\lVert \mathbf{z}\rVert \approx \sqrt{p}$$, und die Toleranz von $$\pm 1$$ schrumpft relativ zur Norm.

Die folgende Abbildung veranschaulicht dies ($$\rho = 0{,}99$$, standardisiert). Shyus Höhenlinien gleichen Scores sind Ellipsen, gestreckt entlang der Major- und gestaucht entlang der Minor-Komponenten. Die Terme von PyOD haben kreisförmige Höhenlinien um die Eigenvektorspitzen, und abseits des Zentrums sieht ihre Summe wie Kreise um das Zentrum aus. Die Anomalien in der Abbildung verletzen die Korrelation, liegen aber nahe am Zentrum. Ihre Norm beträgt höchstens 1,6, während 10 % der Inlier entlang der Major-Achse weiter außen liegen. Der Schwellenwert von PyOD wird daher durch diese Inlier bestimmt, sodass alle Anomalien innerhalb liegen und keine erkannt wird. Unsere Korrektur (d. h. die quadrierte Mahalanobis-Distanz) normiert jede Richtung mit ihrem Eigenwert, bevor der Abstand gemessen wird. Das ergibt Ellipsen entlang der Daten und erkennt 26 von 30 Anomalien.

<div class="row mt-3">
    <div class="col-sm-12 mx-auto">
        {% include figure.liquid path="assets/img/2027-04-30-pca-pyod/projection_problem.png" class="img-fluid" %}
    </div>
</div>
<div class="caption">
    Anomalien nahe dem Zentrum, die die Korrelation verletzen. Links: PyOD 3.6.8 (ROC AUC 0,478, 0 von 30 Anomalien erkannt). Rechts: unsere Korrektur (ROC AUC 0,992, 26 von 30). <a href="https://anonymous.4open.science/r/iclr_2027_blogpost_pca_code-5E5D/blog_post.ipynb">blog_post.ipynb, Abschnitt 2</a>.
</div>

<a id="problem-2"></a>
### Problem 2: Das Vorzeichen

scikit-learn berechnet die Komponenten über eine Singulärwertzerlegung der standardisierten Daten. Die resultierenden Komponenten sind nur bis auf ihr Vorzeichen bestimmt. Ist $$\mathbf{e}_i$$ ein normierter Eigenvektor von $$\mathbf{R}$$, so ist es auch $$-\mathbf{e}_i$$. scikit-learn legt das Vorzeichen per Konvention fest (`svd_flip`) <d-cite key="sklearnPCA"></d-cite>; das ist eine rein rechentechnische Wahl und trägt keine Information über die Daten <d-cite key="broResolvingSignAmbiguity2008"></d-cite>. Das Ausmultiplizieren des Zählers im Score von PyOD ([Anhang C](#appendix-c)) zeigt, warum das wichtig ist:

<p>
$$
\lVert \mathbf{z} - s_i \mathbf{e}_i \rVert = \sqrt{\lVert \mathbf{z}\rVert^2 - 2\, s_i\, y_i + 1}.
$$
</p>

Shyus Score enthält $$y_i^2$$, sodass sich das Vorzeichen von $$\mathbf{e}_i$$ aufhebt. Im Term $$s_i y_i$$ von PyOD spielt das Vorzeichen dagegen eine Rolle. Ein Vorzeichenwechsel verändert die Scores und kann daher die Rangfolge der Beobachtungen ändern, wie bereits in Issue #217 <d-cite key="pyodIssue217"></d-cite> gemeldet.

**Folgen für Anomalien**\
Jeder Term im Score von PyOD misst, wie weit eine Beobachtung von der Spitze eines Eigenvektors $$s_i \mathbf{e}_i$$ entfernt ist, und diese Spitze liegt nur auf einer Seite des Zentrums. Betrachten wir zwei Beobachtungen, die entlang $$\mathbf{e}_i$$ gleich weit vom Zentrum abweichen, aber in entgegengesetzte Richtungen. Die Beobachtung auf der Seite der Spitze rückt näher an sie heran, ihr Term wird also kleiner. Die Beobachtung auf der anderen Seite entfernt sich von ihr, ihr Term wird größer. Für Shyu et al. sind beide Beobachtungen gleich anomal. Für PyOD wirkt eine von ihnen weniger anomal – und welche, entscheidet die Vorzeichenkonvention, nicht die Daten.

Der Effekt ist für die Minor-Komponenten am stärksten, weil ihre Terme die größten Gewichte $$p/\lambda_i$$ tragen. Im Extremfall liegt eine Beobachtung genau auf der Spitze, $$\mathbf{z} = s_i \mathbf{e}_i$$. Für Shyu et al. ist dies vermutlich eine Anomalie mit Score $$1/\lambda_i$$. Für PyOD ist ihr Abstand zur Spitze null, sodass ausgerechnet der Term mit dem größten Gewicht verschwindet.

Die folgende Abbildung veranschaulicht dies anhand zweier korrelierter Merkmale ($$\rho = 0{,}9$$) mit 15 Anomalien, die die Korrelation verletzen. Mit dem Score von PyOD (links) sind die Höhenlinien Kreise, deren Mittelpunkt vom Schwerpunkt in Richtung der Spitze $$s_2\mathbf{e}_2$$ des Minor-Eigenvektors verschoben ist, hier nach rechts unten. Anomalien auf dieser Seite liegen nahe an der Spitze, sodass ihre Scores unter dem Entscheidungsschwellenwert bleiben. Anomalien auf der gegenüberliegenden Seite sind weiter von der Spitze entfernt und werden eher erkannt. Welche Seite bevorzugt wird, hängt allein von der Vorzeichenkonvention ab. Mit unserer [unten](#fix) beschriebenen Korrektur (rechts) sind die Höhenlinien Ellipsen mit Mittelpunkt im Schwerpunkt, sodass Anomalien auf beiden Seiten gleich behandelt werden.

<div class="row mt-3">
    <div class="col-sm-12 mx-auto">
        {% include figure.liquid path="assets/img/2027-04-30-pca-pyod/sign_problem.png" class="img-fluid" %}
    </div>
</div>
<div class="caption">
    Links: PyOD 3.6.8 (ROC AUC 0,754, 6 von 15 Anomalien erkannt), Höhenlinien in Richtung der Spitze des Minor-Eigenvektors verschoben. Rechts: unsere Korrektur (ROC AUC 0,998, 15 von 15), zentrierte Ellipsen. Code: <a href="https://anonymous.4open.science/r/iclr_2027_blogpost_pca_code-5E5D/blog_post.ipynb">blog_post.ipynb, Abschnitt 1</a>.
</div>

### Ein kurzes Beispiel

Betrachten wir $$p = 2$$ standardisierte Merkmale mit Korrelation $$\rho = 0{,}95$$, sodass $$\lambda_1 = 1{,}95$$, $$\mathbf{e}_1 = \tfrac{1}{\sqrt 2}(1, 1)$$ und $$\lambda_2 = 0{,}05$$, $$\mathbf{e}_2 = \tfrac{1}{\sqrt 2}(1, -1)$$. Im Klassifikator von Shyu gilt $$q = r = 1$$. Wir bewerten einige Beobachtungen mit den Statistiken von Shyu und mit PyOD unter beiden Vorzeichen von $$\mathbf{e}_2$$:

| Beobachtung | Beschreibung | PyOD, $$s_2 = +1$$ | PyOD, $$s_2 = -1$$ | Soft (Shyu) |
|---|---|---|---|---|
| $$0$$ | Trainingsmittelwert | 41,03 | 41,03 | 0,00 |
| $$A = (2, 2)$$ | folgt der Korrelation | 121,88 | 121,88 | 4,10 |
| $$B = (1{,}5;\ -1{,}5)$$ | verletzt die Korrelation | 47,26 | 127,26 | 90,00 |
| $$\mathbf{e}_2$$ | Spitze des Minor-Eigenvektors | 1,45 | 81,45 | 20,00 |
| $$-\mathbf{e}_2$$ | ihr Spiegelbild | 81,45 | 1,45 | 20,00 |

Für Shyu et al. ist $$B$$ mit Abstand die anomalste Beobachtung, und $$A$$ ist unauffällig. PyOD mit $$s_2 = +1$$ stuft $$A$$ deutlich höher ein als $$B$$, weil $$A$$ weiter vom Zentrum entfernt liegt ([Problem 1](#problem-1)). Ein Vorzeichenwechsel von $$\mathbf{e}_2$$ kehrt die Reihenfolge von $$A$$ und $$B$$ um und vertauscht die Scores der beiden Spitzen ([Problem 2](#problem-2)). Diejenige Spitze, die mit $$s_2 \mathbf{e}_2$$ zusammenfällt, erhält einen niedrigeren Score als der Trainingsmittelwert selbst.

<div class="row mt-3">
    <div class="col-sm-12 mx-auto">
        {% include figure.liquid path="assets/img/2027-04-30-pca-pyod/blog_post_example_scores.png" class="img-fluid" %}
    </div>
</div>
<div class="caption">
    \(A\) und \(B\) mit den Schwellenwerten beider Eigenvektoren. Dargestellt ist hier \(\mathbf{e}_2\). Code: <a href="https://anonymous.4open.science/r/iclr_2027_blogpost_pca_code-5E5D/blog_post.ipynb">blog_post.ipynb, Abschnitt 3</a>.
</div>

<a id="fix"></a>
### Die Korrektur
Unsere Korrektur übernimmt die in PR #211 <d-cite key="pyodPR211"></d-cite> vorgeschlagene Projektion, normiert aber mit den Eigenwerten $$\lambda_i$$ statt mit den Explained Variance Ratios. Gegenüber PyOD 3.6.8 lautet die Änderung:
```diff
- self.w_components_ = self.detector_.explained_variance_ratio_
+ self.w_components_ = self.detector_.explained_variance_
  ...
- return np.sum(cdist(X, self.selected_components_)
-               / self.selected_w_components_, axis=1).ravel()
+ projections = (X - self.detector_.mean_) @ self.selected_components_.T
+ return np.sum(projections ** 2 / self.selected_w_components_,
+               axis=1).ravel()
```
Damit sind beide Probleme behoben. Die Normierung jeder Hauptrichtung mit ihrem Eigenwert $$\lambda_i$$ stellt sicher, dass die Korrelationsstruktur erhalten bleibt ([Problem 1](#problem-1)). Da $$y_i$$ quadriert in die Statistik eingeht, ist der Anomalie-Score invariant gegenüber dem Vorzeichen der Eigenvektoren ([Problem 2](#problem-2)). Mit den Standardparametern entspricht der Score der quadrierten Mahalanobis-Distanz, also Aggarwals Soft PCA. Für `n_components` $$= p$$ und `n_selected_components` $$= r$$ entspricht er der Minor-Statistik $$T_{\text{minor}}$$ von Shyu et al.

Das folgende Video vergleicht den alten Score (links, PyOD 3.6.8) mit unserer Korrektur (rechts) in 3D, während nach und nach Datenpunkte hinzukommen (einige davon sind Anomalien).

<video controls playsinline preload="metadata" style="width: 100%; height: auto;">
    <source
      src="{{ '/assets/img/2027-04-30-pca-pyod/pca_animation.mp4' | relative_url }}"
      type="video/mp4"
    >
  Ihr Browser unterstützt das Video-Tag nicht.
</video>
<div class="caption">
    Links: PyOD 3.6.8. Rechts: unsere Korrektur. Code: <a href="https://anonymous.4open.science/r/iclr_2027_blogpost_pca_code-5E5D/blog_post.ipynb">blog_post.ipynb, Abschnitt 4</a>.
</div>

Die kugelförmigen Höhenflächen von PyOD sind in Richtung der Spitze des Minor-Eigenvektors $$\mathbf{e}_3$$ verschoben, sodass Anomalien, die die Korrelation verletzen, als normal klassifiziert werden. Die ellipsoiden Höhenflächen unserer Korrektur folgen der Form der Daten und markieren diese Anomalien korrekt.

Der korrigierte Detektor steht als Drop-in-Ersatz für die PCA von PyOD in [pca_new.py](https://anonymous.4open.science/r/iclr_2027_blogpost_pca_code-5E5D/pca_new.py) zur Verfügung. TODO: PR ergänzen, sobald fertig. Außerdem haben wir die irreführenden Kommentare korrigiert und Unit-Tests ergänzt, die die Vorzeicheninvarianz und die exakte Äquivalenz zur Mahalanobis-Distanz prüfen. Gemäß den Beitragsrichtlinien des Projekts werden wir einen Pull Request gegen PyOD öffnen, sobald das zugehörige Issue TODO zugewiesen wurde.

<a id="impact"></a>
## Wie weit es sich verbreitet hat

PyOD wurde mehr als 46 Millionen Mal heruntergeladen <d-cite key="yzhao062"></d-cite>. Um abzuschätzen, wie weit sich der abweichende Score verbreitet hat, haben wir nach Code gesucht, der den Detektor verwendet, und nach Papern, die seine Ergebnisse berichten.

Mithilfe der GitHub Code Search API <d-cite key="githubCodeSearchAPI"></d-cite> haben wir nach allen Importvarianten der PCA von PyOD gesucht, etwa `from pyod.models.pca import PCA` oder `import pyod.models.pca`. Am 10. November 2026 lieferte dies 910 eindeutige Dateien in 525 Repositories, Forks von PyOD ausgenommen. Sie reichen von kleinen Projekten bis zu weit verbreiteten Produkten wie PostHog (40.000 Sterne) oder TDengine (25.000 Sterne).

| # | Repository | Sterne | Beschreibung |
|---|---|---:|---|
| 1 | [PostHog/posthog](https://github.com/PostHog/posthog) | 40727 | :hedgehog: PostHog is the leading platform for building self-driving products |
| 2 | [taosdata/TDengine](https://github.com/taosdata/TDengine) | 25188 | High-performance, scalable time-series database designed for Industrial IoT |
| 3 | [pycaret/pycaret](https://github.com/pycaret/pycaret) | 9860 | Open-source, low-code AutoML platform for Python|
| 4 | [datamllab/tods](https://github.com/datamllab/tods) | 1667 | TODS: An Automated Time-series Outlier Detection System |
| 5 | [Minqi824/ADBench](https://github.com/Minqi824/ADBench) | 1033 | Official Implement of "ADBench: Anomaly Detection Benchmark", NeurIPS 2022. |

Das fünfte Repository in dieser Tabelle ist die offizielle Implementierung von ADBench, einem Benchmark für Anomalieerkennung, der auf der NeurIPS 2022 veröffentlicht wurde <d-cite key="NEURIPS2022_cf93972b"></d-cite>. Han et al. evaluieren 30 Algorithmen auf 57 Datensätzen, darunter die PCA von PyOD mit Standardparametern <d-cite key="adbenchPyODWrapper"></d-cite>. Wir haben die PCA-Baseline gemäß dem ADBench-Protokoll erneut ausgeführt und mit dem korrigierten Detektor verglichen. Unsere PyOD-Ergebnisse stimmen eng mit den veröffentlichten überein <d-cite key="adbenchSupplement"></d-cite>. Protokoll, Reproduktion und vollständige Ergebnisse sind in der [README unseres Codes](https://anonymous.4open.science/r/iclr_2027_blogpost_pca_code-5E5D/adBench/README.md) beschrieben.

<div class="row mt-3">
    <div class="col-sm-12 mx-auto">
        {% include figure.liquid path="assets/img/2027-04-30-pca-pyod/adbench_pca_diff_landscape.png" class="img-fluid" %}
    </div>
</div>
<div class="caption">
    Änderung von ROC AUC und AP je ADBench-Datensatz (Korrektur − PyOD 3.6.8), gemittelt über Seeds, 51 vergleichbare Datensätze. Grün: Korrektur besser, Rot: Korrektur schlechter. Code: <a href="https://anonymous.4open.science/r/iclr_2027_blogpost_pca_code-5E5D/adBench/run_adbench_pca.py">adBench/run_adbench_pca.py</a>.
</div>

Mit aktuellen Bibliotheksversionen scheitert die PCA von PyOD auf sechs der Datensätze, weil Eigenwerte von null zu unendlichen Scores führen; wir vergleichen beide Detektoren daher auf den verbleibenden 51. Im Mittel steigt die ROC AUC von 0,718 auf 0,744, während die Average Precision von 0,357 auf 0,320 sinkt. Keiner der beiden Unterschiede ist signifikant (zweiseitiger Wilcoxon-Vorzeichen-Rang-Test <d-cite key="wilcoxon1945individual"></d-cite>). Die Änderungen pro Datensatz sind jedoch in beide Richtungen groß. So steigt die ROC AUC für vowels von 0,656 auf 0,912, während die AP für satimage-2 von 0,858 auf 0,436 sinkt.

ADBench pinnt `pyod==1.0.0` <d-cite key="adbenchRequirements"></d-cite>, dessen Scoring-Code identisch mit `3.6.8` ist <d-cite key="pyodPCA100"></d-cite>. Die PCA-Ergebnisse von ADBench sind daher reproduzierbar, der Code dahinter ist aber nicht verifiziert. Sie wurden nicht mit der Methode von Shyu et al. berechnet, die PyOD zitiert. Die Zahlen sind eine korrekte Messung der Implementierung von PyOD, werden aber unter einer Methode berichtet, deren zitierter Definition sie nicht folgen.

ADBench wurde 865-mal zitiert <d-cite key="NEURIPS2022_cf93972b"></d-cite> und das PyOD-Paper 1484-mal <d-cite key="zhao2019pyod"></d-cite>, wobei nicht jede zitierende Arbeit den PCA-Detektor verwendet. Wir haben keinen verlässlichen Weg gefunden, die Paper zu zählen, die die PCA-Implementierung von PyOD nutzen, konnten aber mehrere Beispiele manuell bestätigen, etwa „Conformal prediction interval for dynamic time-series“ (416 Zitationen) <d-cite key="xuConformalPredictionInterval2021"></d-cite>, PIDForest (49 Zitationen) <d-cite key="PIDForestAnomalyDetection2019"></d-cite> und UADB (24 Zitationen) <d-cite key="yeUADBUnsupervisedAnomaly2023"></d-cite>. Alle Zitationszahlen wurden am 15. November 2026 aus Google Scholar abgerufen.

Die Prüfung des Codes eines Papers findet nur direkte Nutzung. Ein Paper, das PCA-Ergebnisse aus einem Benchmark wie ADBench übernimmt, berichtet betroffene Zahlen, ohne PyOD jemals zu importieren, und sein Repository – falls es eines gibt – gibt keinen Hinweis darauf. Selbst die wenigen oben genannten Fälle erforderten, den Code Repository für Repository zu lesen.

Derzeit gibt es keinen systematischen Weg, die Frage zu beantworten: Welche veröffentlichten Ergebnisse hängen von diesem Code ab?

## Warum es überlebt hat

Damit sollen nicht die Maintainer von PyOD beschuldigt werden. Eine Bibliothek dieser Reichweite ist ein enormer Dienst an der Community, und wir nutzen sie selbst. Fehler passieren in jeder Codebasis, auch in unserer. Uns interessiert, wie dieser eine Fehler so lange überleben konnte. Drei Beobachtungen stechen hervor.

**Die Dokumentation zitiert die Methode, der Code implementiert sie nicht.**
Als der Detektor in Commit [a6f05f3b](https://github.com/yzhao062/pyod/commit/a6f05f3be9cddeea2e5a57f6c9c7cbb05efd9723) hinzugefügt wurde, verwies sein Docstring bereits auf Shyu et al. <d-cite key="shyu2003"></d-cite> und Aggarwal <d-cite key="aggarwal2017"></d-cite>, gab aber auch die tatsächliche Berechnung an. Einen Tag später ersetzte Commit [fa7f9fc](https://github.com/yzhao062/pyod/commit/fa7f9fc948717c0d30ee95a247ae79ea5398f9d5) diese Formel durch die Beschreibung eines Abstands zu der von den Eigenvektoren aufgespannten Hyperebene, die den zitierten Methoden näher ist als dem Code. Der Code selbst blieb unverändert. Wer die Dokumentation liest, hat daher guten Grund anzunehmen, dass der Detektor die zitierte Methode implementiert. Solche Abweichungen zwischen Kommentaren und Code sind verbreitet <d-cite key="wen2019codecomment"></d-cite>.

**Das Problem wurde gemeldet, aber nicht behoben.**
2019 beobachtete ein Nutzer auf seinen Daten, dass die PCA-Scores mit dem euklidischen Abstand zum Zentrum korrelieren ($$r = 0{,}99$$) – das ist [Problem 1](#problem-1). Der Maintainer schloss das Issue und führte dies auf die Daten zurück <d-cite key="pyodIssue64"></d-cite>. 2020 implementierte Pull Request #211 <d-cite key="pyodPR211"></d-cite> die korrekte Projektion, und Issue #217 <d-cite key="pyodIssue217"></d-cite> wies darauf hin, dass die Rangfolge vom Vorzeichen der Komponenten abhängt – das ist [Problem 2](#problem-2). 2021 griff Pull Request #340 <d-cite key="pyodPR340"></d-cite> die Idee von #217 auf. Keiner dieser Vorschläge wurde bis heute gemergt, und jeder von ihnen weicht auf seine eigene Weise von der zitierten Methode ab ([Anhang D](#appendix-d)).

**Die Verbreitung ging weiter.**
Obwohl die Abweichung seit 2019 öffentlich war, übernahm ADBench den Detektor 2022 als Baseline. Von dort aus verbreiteten sich seine Ergebnisse in Paper, die Benchmark-Zahlen übernehmen, ohne PyOD jemals zu importieren.

Ein Problem zu melden reicht daher nicht aus. Die Meldung liegt in einem Issue-Tracker, die betroffenen Ergebnisse stehen in Papern, und nichts verbindet beides.

## Was fehlt

Dieser Fall ist nicht auf PyOD beschränkt. Kriegel et al. <d-cite key="kriegel2017black"></d-cite> zeigten für Laufzeitevaluationen, dass Experimente immer auch Implementierungen und nicht nur Algorithmen vergleichen und dass die Implementierung allein die gemessenen Unterschiede dominieren kann. Engstrom et al. stellten fest, dass undokumentierte Implementierungsdetails einen Großteil des berichteten Vorteils eines populären Deep-Reinforcement-Learning-Algorithmus ausmachen <d-cite key="engstromImplementationMattersDeep2020"></d-cite>. Leitfäden zu Fallstricken im maschinellen Lernen helfen Autor:innen, solche Fehler vor der Veröffentlichung zu vermeiden <d-cite key="lones2024pitfalls"></d-cite>. Eine Abweichung von der zitierten Methode ist jedoch schwer zu entdecken, da sie keinen Fehler auslöst <d-cite key="tambon2024silent"></d-cite> und die korrekte Ausgabe oft unbekannt ist <d-cite key="kanewala2014testing"></d-cite>. Keiner dieser Ansätze deckt ab, was mit veröffentlichten Ergebnissen geschieht, sobald eine Abweichung bekannt wird.

Eine Korrektur in der Bibliothek erreicht veröffentlichte Ergebnisse nicht, doch deren gepinnte Versionen machen es möglich festzustellen, ob sie betroffen sind. Dazu braucht es eine Aufzeichnung darüber, welcher Code und welche Versionen betroffen sind, eine Aufzeichnung darüber, von welchem Code und welchen Versionen ein Ergebnis abhängt, und eine Möglichkeit, beides zu verknüpfen. Bestehende Praktiken liefern davon nur Teile. Gepinnte Versionen halten die Abhängigkeiten fest, aber es gibt nichts, womit man sie abgleichen könnte. Changelogs erfassen nur behobene Probleme und verknüpfen sie nicht in maschinenlesbarer Form mit konkreten Funktionen und Versionen. Issue-Tracker beschreiben offene Probleme, sind aber über Repositories verstreut und nicht mit betroffenen Versionen verknüpft. Jedes Ergebnis erneut auszuführen würde den Abgleich überflüssig machen, verlagerte die Kosten aber auf alle Leser:innen jedes Papers.

Selbst wo Infrastruktur existiert, ist es schwierig, die Aufzeichnungen zu korrigieren. Zurückgezogene Datensätze zirkulieren über Derivate weiter <d-cite key="peng2021mitigating"></d-cite>, und obwohl es Standards für Rückzugsmitteilungen (Retraction Notices) gibt <d-cite key="niso2024crec"></d-cite>, gelten sie für ganze Paper oder Datensätze, nicht für einzelne Ergebnisse, die von einer Bibliothek betroffen sind.

Für Sicherheitslücken existiert diese Infrastruktur. CVE entstand, weil jedes Tool seine eigene, inkompatible Liste von Schwachstellen führte <d-cite key="mann1999cve"></d-cite>. Heute erfasst das OSV-Schema betroffene Paketversionen in einem maschinenlesbaren Format und wird unter anderem von GitHub Security Advisories und der PyPI Advisory Database exportiert <d-cite key="osvschema"></d-cite>. Tools wie pip-audit <d-cite key="pipAudit"></d-cite> und Dependabot <d-cite key="dependabot"></d-cite> gleichen die Abhängigkeiten eines Projekts automatisch mit diesen Einträgen ab, auch wenn Einträge allein noch kein Handeln sicherstellen <d-cite key="kula2018developers"></d-cite>. In der IT-Sicherheit ist eine veröffentlichte Schwachstelle kein Grund zur Scham. Bekannte Abweichungen sollten genauso behandelt werden: Entscheidend ist nicht, dass sie aufgetreten sind, sondern dass sie sichtbar und behoben werden.

## Ein Vorschlag für eine Community-weite Lösung
Wir schlagen Correctness Advisories vor, die eine bekannte Abweichung in einer Bibliothek mit den veröffentlichten Ergebnissen verknüpfen, die sie betrifft. Die meisten der benötigten Bausteine existieren bereits. Coordinated Flaw Disclosure beschreibt, wie Fehler in eingesetzten Machine-Learning-Modellen gemeldet und behoben werden <d-cite key="cattell2024cfd"></d-cite>. Die AI Vulnerability Database trennt eine allgemeine Schwachstelle von den einzelnen Meldungen ihres Auftretens <d-cite key="avid"></d-cite>. Beide richten sich jedoch an eingesetzte Modelle und deren Anbieter, nicht an Bibliotheken oder die veröffentlichten Ergebnisse, die von ihnen abhängen. Um diese Lücke zu schließen, schlagen wir vier Bausteine vor.

**Provenienz**
Repositories würden exakte Versionen pinnen, zum Beispiel in einem Lockfile. Paper würden für jede Baseline angeben, ob sie berechnet (Bibliothek und Version) oder übernommen (Quelle) wurde <d-cite key="smith2016software"></d-cite>. Ergebnisse zu übernehmen bliebe legitim, würde aber nachvollziehbar.

**Advisories**
Wir schlagen vor, Advisories am OSV-Schema <d-cite key="osvschema"></d-cite> auszurichten. Sie würden die OSV-Felder für Identifikatoren, betroffene Versionen und Referenzen übernehmen, den Sicherheits-Schweregrad durch die Auswirkung auf Ergebnisse ersetzen und die Art der Abweichung, die zitierte Methode und einen Status ergänzen. Ein Advisory könnte zudem die betroffenen Funktionen benennen, sodass beim Abgleich geprüft werden kann, ob ein Projekt sie tatsächlich aufruft, wie es die Go Vulnerability Database tut <d-cite key="govulndb"></d-cite>. Für PyOD könnte ein Advisory so aussehen:
```yaml
id: MLCA-2026-0001
modified: "2026-11-18T00:00:00Z"
summary: PCA score deviates from the cited methods
type: deviation-from-cited-method
cited_method: [Shyu et al. 2003, Aggarwal 2017]
impact: ranking          # values | ranking | classification
status: confirmed        # reported | confirmed | disputed | fixed
affected:
  - package: {ecosystem: PyPI, name: pyod}
    ranges: [{type: ECOSYSTEM, events: [{introduced: "0"}]}]   # kein "fixed"-Event: noch offen
    symbols: [pyod.models.pca.PCA.decision_function]
references:
  - {type: REPORT, url: "https://github.com/yzhao062/pyod/issues/64"}   # weitere: #217, #211, #340
```

Da ein Advisory viele Ergebnisse betreffen kann, die zu unterschiedlichen Zeitpunkten und mit unterschiedlicher Sicherheit identifiziert werden, schlagen wir für jedes betroffene Ergebnis einen eigenen Eintrag vor. Sein Status könnte VEX <d-cite key="cisa2023vex"></d-cite> folgen, wobei `not_affected` eine Begründung erfordert:
```yaml
id: MLCR-2026-0001
advisory: MLCA-2026-0001
result: ADBench (NeurIPS 2022), PCA results in Fig. 4a/c, Fig. 5a–d, Fig. 7a/d, and Appx. D
status: affected       # under_investigation | affected | not_affected (+ Begründung) | fixed
evidence: pins pyod==1.0.0, scoring code identical to 3.6.8
```
Für ADBench ist der Nachweis der identische Scoring-Code <d-cite key="pyodPCA100"></d-cite>. Ein Eintrag bliebe `under_investigation`, bis jemand den öffentlichen Code und das Paper geprüft hat. Diese Prüfung würde einmal durchgeführt und könnte wiederverwendet werden. Advisories und Einträge könnten in einem öffentlichen Git-Repository mit einer Datei pro Eintrag gehostet und über Pull Requests kuratiert werden.

Maintainer könnten beitragen, indem sie eine eigene Changelog-Kategorie für Änderungen einführen, die bei gleichen Daten und Parametern die Ergebnisse verändern, wie sie scikit-learn bereits pflegt <d-cite key="sklearn13changelog"></d-cite>. Solche Einträge ließen sich direkt in Advisories mit einem fixed-Event im betroffenen Versionsbereich übersetzen.

**Abgleich**
Ein Tool nach dem Vorbild von Dependency-Audits wie pip-audit <d-cite key="pipAudit"></d-cite> könnte die Abhängigkeiten eines Repositorys, einschließlich transitiver, mit den betroffenen Versionsbereichen abgleichen. Da Versionsinformationen allein die Zahl der betroffenen Projekte überschätzen <d-cite key="pashchenko2022vuln4real"></d-cite>, sollte das Tool anschließend nur Repositories behalten, die eine betroffene Funktion importieren oder aufrufen <d-cite key="ponta2020detection"></d-cite>. Übernommene Ergebnisse ließen sich über die Provenienzangabe des zitierenden Papers abgleichen. Eine bloße Zitation ergäbe höchstens `under_investigation`. Wir schlagen vor, den Abgleich auf direkte Wiederverwendung zu beschränken, da Ketten übernommener Ergebnisse die Evidenz schnell abschwächen. Ein neues Advisory würde dieselbe Suche in umgekehrter Richtung auslösen, wie wir sie in [Wie weit es sich verbreitet hat](#impact) manuell durchgeführt haben.

**Sichtbarkeit**
Warnungen wirken am besten dort, wo Forschende ohnehin arbeiten: als Dependency-Audit beim Ausführen von Code, als Prüfung in der Literaturverwaltung beim Zitieren eines Papers und als dieselbe Prüfung im Review. Zotero warnt bereits, bevor zurückgezogene Arbeiten zitiert werden <d-cite key="zotero2019retraction"></d-cite>, und da es Plugins von Drittanbietern unterstützt <d-cite key="zoteroPluginDev"></d-cite>, ließe sich eine ähnliche Prüfung anbieten, ohne Zotero selbst zu ändern. Anders als eine Rückzugsmitteilung würde sich eine solche Warnung auf ein bestimmtes Ergebnis beziehen, z. B. die PCA-Ergebnisse in ADBench, nicht auf das Paper als Ganzes.

### Einschränkungen
Unser Vorschlag ist ein erster Entwurf und lässt mehrere Fragen offen.

**Was als Abweichung gilt** Unabhängige Implementierungen desselben Algorithmus liefern oft leicht unterschiedliche Ergebnisse <d-cite key="herbold2023differential"></d-cite>, und die verzerrte Standardabweichung in PyOD hat kaum Auswirkungen. Ein Advisory bräuchte daher ein Kriterium, z. B. dass die Abweichung die Rangfolge in einem einfachen Beispiel ändert. Meinungsverschiedenheiten zwischen Meldenden und Maintainern sind zu erwarten. Coordinated Flaw Disclosure löst sie mit einem Gremium unabhängiger Schiedspersonen, das entscheidet, wenn ein Anbieter eine Meldung zurückweist <d-cite key="cattell2024cfd"></d-cite>, und ein ähnliches Gremium könnte entscheiden, ob eine Abweichung erfasst wird.

**Präzision des Abgleichs** Ein Abgleich über Importe übersieht Modelle, die über ihren Namen instanziiert werden (wie in PyCaret), kopierten Quellcode und unveröffentlichten Code, und bestehende Paper enthalten keine Provenienzangaben. Für die bestehende Literatur wird der Abgleich daher überwiegend Einträge liefern, die eine manuelle Prüfung erfordern.

**Betroffen heißt nicht falsch** Ein bestätigter Eintrag besagt, dass die berichteten Zahlen nicht mit der zitierten Methode gewonnen wurden. Er besagt nicht, dass die Schlussfolgerungen des Papers falsch sind. In ADBench ist PCA eine von 30 Baselines, und ob sich die Gesamtergebnisse ändern, erfordert eine separate Analyse.

**Evaluation** Über die manuelle Suche in [Wie weit es sich verbreitet hat](#impact) hinaus haben wir nicht evaluiert, ob ein automatisierter Abgleich präzise genug ist oder ob Warnungen verändern, wie Forschende Ergebnisse wiederverwenden.

**Nachhaltigkeit** Advisories müssen über viele Jahre gepflegt werden. Datenbanken für Security Advisories zeigen, dass dies möglich ist, es erfordert aber eine Institution, die sie hostet, und nicht nur eine kleine Gruppe Freiwilliger.

## Fazit

Der PCA-Detektor von PyOD zitiert Shyu et al., bewertet Beobachtungen aber mit einer gewichteten Summe euklidischer Abstände zu den Eigenvektorspitzen statt mit varianznormierten Projektionen. Dieser Score hängt vom willkürlichen Vorzeichen jedes Eigenvektors ab und ist näherungsweise proportional zum Abstand vom Zentrum. Der Detektor ist daher weitgehend unempfindlich gegenüber Anomalien, die die Korrelationsstruktur verletzen, ohne weit vom Zentrum entfernt zu liegen – also genau gegenüber den Anomalien, die die Minor-Komponenten erkennen sollen. Ersetzt man die Abstände durch quadrierte, mit den Eigenwerten normierte Projektionen, wird der Score vorzeicheninvariant und zur quadrierten Mahalanobis-Distanz.

Der korrigierte Detektor ist nicht durchweg besser. Auf ADBench ändern sich ROC AUC und AP je nach Datensatz deutlich in beide Richtungen. Unser Hauptbefund ist daher kein Leistungsgewinn. Er besteht darin, dass die PCA-Ergebnisse, die in ADBench, in Papern wie PIDForest und UADB sowie in 525 Repositories berichtet werden, die den Detektor verwenden, mit einem anderen Score gewonnen wurden als dem in den zitierten Referenzen beschriebenen – obwohl die Abweichung seit 2019 gemeldet ist.

Diese Ergebnisse sind reproduzierbar, aber der Code dahinter ist nicht verifiziert. Das bedeutet nicht, dass die Schlussfolgerungen der betroffenen Paper falsch sind, sondern nur, dass ihre PCA-Zahlen nicht von der zitierten Methode stammen. Die Korrektur des Detektors erreicht diese Ergebnisse nicht, und derzeit verbindet nichts eine bekannte Abweichung mit den Ergebnissen, die sie betrifft. Über diesen Fall hinaus haben wir skizziert, wie bekannte Abweichungen so sichtbar gemacht werden könnten wie bekannte Sicherheitslücken, größtenteils durch die Kombination bestehender Bausteine: Coordinated Flaw Disclosure, Advisories nach dem Vorbild von OSV, VEX-artige Einträge für betroffene Ergebnisse, Provenienzangaben in Papern und Warnungen in Tools, die Forschende ohnehin nutzen.

### Ausblick
Unsere Korrektur ändert bewusst nur den Score, damit sich ihr Effekt auf ADBench dieser einen Komponente zuordnen lässt. Die übrigen Teile des PCC, nämlich Trimming, getrennte Major- und Minor-Statistiken und zwei Schwellenwerte, sind in PyOD nicht implementiert. Solche Komponenten können das Verhalten einer Methode erheblich verändern. Ein naheliegender nächster Schritt ist, den vollständigen PCC als eigene Detektorklasse zu implementieren und seine Komponenten einzeln zu evaluieren. Dies erfordert eine zusätzliche Designentscheidung, da der PCC ein Klassifikator mit zwei Schwellenwerten ist, während Benchmarks wie ADBench einen einzelnen kontinuierlichen Score evaluieren.

Für den Vorschlag bleiben Governance, der genaue Status-Workflow und der automatisierte Abgleich offene Designfragen. Die Erfahrungen aus dem Aufbau von CVE sind ein Ausgangspunkt <d-cite key="baker1999cve"></d-cite>.

### Abschließende Gedanken

Popularität ist kein Beweis für Korrektheit <d-cite key="soergel2015rampant"></d-cite>. Eine weit verbreitete Bibliothek wird womöglich sogar weniger geprüft, weil alle annehmen, dass es schon jemand anderes getan hat.

Generative KI wirkt hier in beide Richtungen. Von Menschen geschriebener Code wird samt seiner Fehler zu Trainingsdaten, sodass eine Implementierung wie diese in generiertem Code wieder auftauchen kann <d-cite key="jesseLargeLanguageModels2023"></d-cite>. Gleichzeitig ist der Vergleich einer Implementierung mit ihrer Dokumentation oder dem zitierten Paper genau die Art mühsamer Aufgabe, bei der LLMs helfen können, auch wenn aktuelle Modelle noch mehr als die Hälfte realer Abweichungen zwischen Paper und Code übersehen <d-cite key="baumgartner2026scicoqa"></d-cite>. Wie Shah <d-cite key="researchAskingAuthorsTheir2026"></d-cite> in Erinnerung ruft, sind Autor:innen für die Richtigkeit und Integrität der Paper verantwortlich, die sie unter ihrem Namen einreichen. Wir meinen, dass sich diese Verantwortung auch auf die Werkzeuge erstreckt, auf denen ein Paper aufbaut.

Advisories helfen, sobald ein Problem bekannt ist. Es überhaupt erst zu finden, hängt weiterhin von einzelnen Forschenden ab. Wir empfehlen daher: Bevor Sie Ergebnisse für eine Methode berichten, prüfen Sie, ob ihre Implementierung tut, was das Paper sagt. Lesen Sie die Scoring-Funktion, sehen Sie sich die offenen Issues an und testen Sie sie an einem einfachen Beispiel, dessen Ergebnis Sie kennen. Einen ersten Durchgang kann ein LLM übernehmen.

<a id="appendix-a"></a>
## Anhang A
**Der vollständige Principal Component Classifier**

**Standardisierung**\
Da die Merkmale auf unterschiedlichen Skalen gemessen werden, würden Merkmale mit großen Wertebereichen die Hauptkomponenten dominieren. Jedes Merkmal $$k$$ wird daher mit seinem Stichprobenmittelwert und seiner Stichprobenvarianz über alle $$n$$ Trainingsbeobachtungen $$j$$ standardisiert:

<p>
$$
\bar{x}_k = \frac{1}{n}\sum_{j=1}^{n} x_{jk}, \qquad
s_{kk} = \frac{1}{n-1}\sum_{j=1}^{n} \left(x_{jk} - \bar{x}_k\right)^2
$$
</p>

Der Faktor $$\frac{1}{n-1}$$ liefert eine erwartungstreue Varianzschätzung, da der Mittelwert selbst aus denselben Daten geschätzt wird. Die standardisierte Beobachtung $$\mathbf{z}_j$$ lautet dann

<p>
$$
z_{jk} = \frac{x_{jk} - \bar{x}_k}{\sqrt{s_{kk}}}, \qquad
\mathbf{z}_j = (z_{j1}, z_{j2}, \dots, z_{jp})^\top
$$
</p>

**Trimming**\
Um den PCC robuster gegenüber Anomalien in den Trainingsdaten zu machen, werden die $$\gamma \cdot n$$ Beobachtungen mit den größten Mahalanobis-Distanzen <d-cite key="mahalanobis1936generalized"></d-cite> entfernt, da sie Extremwerte darstellen könnten (z. B. $$\gamma=0{,}005$$ in <d-cite key="shyu2003"></d-cite>). Anschließend werden Mittelwert und Korrelationsmatrix aus den verbleibenden Beobachtungen neu geschätzt.

**Major- und Minor-Komponenten**\
Anstatt über alle Komponenten zu summieren, werten Shyu et al. die Major- und Minor-Komponenten mit zwei separaten Statistiken aus.

- Die Major-Statistik summiert über die ersten $$q$$ Komponenten, wobei $$q$$ die kleinste Anzahl von Komponenten ist, die zusammen mehr als $$50\,\%$$ der Gesamtvarianz erklären. Sie erkennt Beobachtungen mit Extremwerten entlang der dominanten Richtungen.

  <p>
  $$T_{\text{major}}(\mathbf{z}) = \sum_{i=1}^{q} \frac{y_i^2}{\lambda_i}.$$
  </p>

- Die Minor-Statistik summiert über die letzten $$r$$ Komponenten mit Eigenwerten $$\lambda_i < 0{,}20$$ (der Korrelationsmatrix), wobei das Gewicht $$1/\lambda_i$$ kleine Abweichungen verstärkt, die die normale Korrelationsstruktur verletzen.

  <p>
  $$T_{\text{minor}}(\mathbf{z}) = \sum_{i=p-r+1}^{p} \frac{y_i^2}{\lambda_i}.$$
  </p>

Die Minor-Statistik ähnelt Aggarwals *Hard PCA* <d-cite key="aggarwal2017"></d-cite>, die Beobachtungen nur entlang von Komponenten mit geringer Varianz bewertet. Durch die Kombination mit $$T_{\text{major}}(\mathbf{z})$$ erfasst der PCC auch Extremwerte entlang der dominanten Richtungen, die Hard PCA übersieht. Anders als ein einzelner kombinierter Score zeigen die beiden Statistiken außerdem an, welche Art von Anomalie eine Beobachtung darstellt.

**Klassifikation**\
Um eine neue Beobachtung $$\mathbf{z}$$ zu klassifizieren, werden Anomalieschwellenwerte $$c_1$$ und $$c_2$$ für die Major- und die Minor-Statistik festgelegt. Eine Beobachtung wird als anomal markiert, wenn einer der beiden Schwellenwerte überschritten wird:

<p>
$$T_{\text{major}}(\mathbf{z}) > c_1 \quad \text{oder} \quad T_{\text{minor}}(\mathbf{z}) > c_2 .$$
</p>

Die Schwellenwerte $$c_1$$ und $$c_2$$ sind die $$(1 - \alpha_1)$$- und $$(1 - \alpha_2)$$-Quantile der empirischen Verteilungen der beiden Statistiken auf den Trainingsdaten. Sie werden anhand individueller Fehlalarmraten $$\alpha_1$$ und $$\alpha_2$$ gewählt, um eine Gesamtfehlalarmrate von $$\alpha = \alpha_1 + \alpha_2 - \alpha_1 \alpha_2$$ zu erreichen, unter der Annahme, dass die beiden Statistiken unabhängig sind.

<a id="appendix-b"></a>
## Anhang B
<a id="pyods_pca"></a>
**Überblick über die Unterschiede zwischen PCC und PyOD**

| Schritt | Shyu et al. | PyOD (Standardwerte) | Auswirkung |
|---|---|---|---|
| Standardisierung | Stichproben-Standardabweichung (`ddof=1`) | `StandardScaler` (`ddof=0`) | vernachlässigbar |
| Zerlegung | Eigenwertzerlegung von $$\mathbf{R}$$ | SVD der standardisierten Daten | identisch bis auf das Vorzeichen jeder Komponente |
| Trimming | Entfernen der $$\gamma \cdot n$$ extremsten Beobachtungen ($$\gamma = 0{,}005$$) | keines | weniger robust gegenüber Anomalien in den Trainingsdaten |
| Komponenten | $$q$$ Major- und $$r$$ Minor-Komponenten, getrennt ausgewertet | alle $$p$$ Komponenten in einer einzigen Summe | Major- und Minor-Anomalien werden nicht unterschieden |
| Schwellenwert | $$c_1$$, $$c_2$$ aus den Fehlalarmraten $$\alpha_1$$, $$\alpha_2$$ | ein Quantil der Trainingsscores (`contamination=0.1`) | ein einzelner Schwellenwert |
| Score | $$\sum_i \frac{y_i^2}{\lambda_i}$$ pro Statistik | $$p \sum_i \frac{\lVert \mathbf{z} - s_i \mathbf{e}_i \rVert}{\lambda_i}$$ | andere Funktion (siehe [Was PyOD berechnet](#what-pyod-computes)) |

**Standardisierung**\
Der `StandardScaler` <d-cite key="standardscaler"></d-cite> teilt durch die verzerrte Standardabweichung (`ddof=0`), was laut scikit-learn kaum Einfluss auf die Modellleistung hat.

**Zerlegung**\
scikit-learn, das von PyOD verwendet wird, berechnet die Komponenten über eine Singulärwertzerlegung der standardisierten Daten statt über eine Eigenwertzerlegung der Korrelationsmatrix. Beide liefern dieselben Komponenten, allerdings nur bis auf ihr Vorzeichen: Ist $$\mathbf{e}_i$$ ein normierter Eigenvektor von $$\mathbf{R}$$, so ist es auch $$-\mathbf{e}_i$$. Implementierungen legen das Vorzeichen daher per Konvention fest, etwa mit `svd_flip` in scikit-learn <d-cite key="sklearnPCA"></d-cite>, und das resultierende Vorzeichen hat keine sinnvolle Interpretation in Bezug auf die Daten <d-cite key="broResolvingSignAmbiguity2008"></d-cite>. Für den Score von Shyu ist das irrelevant, da $$y_i$$ quadriert in die Statistik eingeht. Für den Score von PyOD dagegen nicht, wie in [Problem 2](#problem-2) gezeigt.

**Trimming**\
PyOD fittet die PCA auf allen Trainingsbeobachtungen. Anomalien in nominell normalen Trainingsdaten können die geschätzten Varianzen aufblähen und die Komponenten drehen <d-cite key="shyu2003"></d-cite>, sodass Anomalien teilweise Teil des gelernten normalen Unterraums werden <d-cite key="ringbergSensitivityPCATraffic2007a"></d-cite>. Shyu et al. verhindern dies durch Trimming.

**Komponentenauswahl**\
Standardmäßig verwendet PyOD alle $$p$$ Komponenten. Wird `n_components` auf $$k < p$$ gesetzt, werden nur die $$k$$ Komponenten mit der größten Varianz behalten. `n_selected_components` wählt dann die *letzten* davon für das Scoring aus. Sie entsprechen nur für $$k = p$$ Minor-Komponenten im Sinne von Shyu et al. PyOD bietet keine Option, Major- und Minor-Komponenten getrennt auszuwerten.

<a id="appendix-c"></a>
## Anhang C

**Ausmultiplizieren des Abstands zu einer Eigenvektorspitze**

Norm als Skalarprodukt

<p>
$$\lVert \mathbf{z} - s_i \mathbf{e}_i \rVert^2 = (\mathbf{z} - s_i \mathbf{e}_i)^\top(\mathbf{z} - s_i \mathbf{e}_i)$$
</p>

Ausmultiplizieren; da das Skalarprodukt symmetrisch ist, lassen sich die beiden mittleren Terme zusammenfassen

<p>
$$
\begin{aligned}
(\mathbf{z} - s_i \mathbf{e}_i)^\top(\mathbf{z} - s_i \mathbf{e}_i)
&= \mathbf{z}^\top\mathbf{z} \;-\; s_i\, \mathbf{z}^\top\mathbf{e}_i \;-\; s_i\, \mathbf{e}_i^\top\mathbf{z} \;+\; s_i^2\, \mathbf{e}_i^\top\mathbf{e}_i \\
&= \mathbf{z}^\top\mathbf{z} \;-\; 2\, s_i\, \mathbf{e}_i^\top\mathbf{z} \;+\; s_i^2\, \mathbf{e}_i^\top\mathbf{e}_i
\end{aligned}
$$
</p>

Nun setzen wir ein

| Term | Wert | Begründung |
|---|---|---|
| $$\mathbf{z}'\mathbf{z}$$ | $$\lVert \mathbf{z} \rVert^2$$ | Definition der Norm |
| $$\mathbf{e}_i'\mathbf{z}$$ | $$y_i$$ | Shyu et al., Gl. (4): $$y_i = \mathbf{e}_i'\mathbf{z}$$ |
| $$s_i^2$$ | $$1$$ | $$s_i \in \{-1, +1\}$$ |
| $$\mathbf{e}_i'\mathbf{e}_i$$ | $$1$$ | Eigenvektoren haben die Länge eins (`components_` ist orthonormal) |

Damit ergibt sich:

<p>
$$\lVert \mathbf{z} - s_i \mathbf{e}_i \rVert^2 = \lVert \mathbf{z} \rVert^2 - 2\,s_i\,y_i + 1$$
</p>

Ziehen der Wurzel ergibt

<p>
$$\lVert \mathbf{z} - s_i \mathbf{e}_i \rVert = \sqrt{\lVert \mathbf{z} \rVert^2 - 2\,s_i\,y_i + 1}$$
</p>

<a id="appendix-d"></a>
## Anhang D

**In früheren Meldungen vorgeschlagene Scores**

2020 implementierte Pull Request #211 <d-cite key="pyodPR211"></d-cite> die korrekte Projektion, normierte die quadrierten Scores aber mit dem Explained Variance Ratio statt mit dem Eigenwert $$\lambda_i$$. Dadurch werden alle Scores mit der Gesamtvarianz $$\sum_j \lambda_j$$ skaliert; die exakte Mahalanobis-Rangfolge bleibt erhalten, während sich die Zahlenwerte ändern. Mit dem Standardwert `whiten=False` ist der Vorschlag in Issue #217 <d-cite key="pyodIssue217"></d-cite> äquivalent zu #211. Mit `whiten=True` liefert `fit_transform` bereits normierte Scores $$t_i = y_i / \sqrt{\lambda_i}$$, sodass jede Komponente mit $$1/\lambda_i^2$$ statt mit $$1/\lambda_i$$ gewichtet wird.

2021 griff Pull Request #340 <d-cite key="pyodPR340"></d-cite> die Idee von #217 auf, zentriert aber mit dem Mittelwert des aktuellen Batches statt mit dem Trainingsmittelwert. Dadurch erhält eine einzeln bewertete Testbeobachtung immer den Score 0. Außerdem steht das Gewicht innerhalb des Quadrats, sodass die Komponenten mit $$1/\lambda_i^2$$ statt mit $$1/\lambda_i$$ gewichtet werden.

Die folgende Tabelle fasst die Scores zusammen.

| Version | Score |
|---|---|
| aktuelles PyOD | $$\sum_i \frac{\lVert \mathbf{z} - \mathbf{e}_i \rVert_2}{\lambda_i / \sum_j \lambda_j}$$|
| [#211](https://github.com/yzhao062/pyod/pull/211) | $$\sum_i \frac{y_i^2}{\lambda_i / \sum_j \lambda_j}$$|
| [#217](https://github.com/yzhao062/pyod/issues/217) | $$\sum_i \frac{t_i^2}{\lambda_i / \sum_j \lambda_j}$$|
| [#340](https://github.com/yzhao062/pyod/pull/340) | $$\sum_i \left(\frac{t_i - \bar{t}_i^{\text{batch}}}{\lambda_i / \sum_j \lambda_j}\right)^2$$ |
| unsere Korrektur | $$\sum_i \frac{y_i^2}{\lambda_i}$$ |

wobei $$y_i = \mathbf{e}_i^\top \mathbf{z}$$ die Projektion der standardisierten Beobachtung $$\mathbf{z}$$ auf den Eigenvektor $$\mathbf{e}_i$$ bezeichnet und $$t_i$$ die Ausgabe von `transform` in sklearn ($$t_i = y_i$$ für `whiten=False` und $$t_i = y_i / \sqrt{\lambda_i}$$ für `whiten=True`).
