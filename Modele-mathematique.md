# Modèle mathématique

Document de référence d'ImmoDécision. Il décrit ce que l'outil calcule, dans
quel ordre, et sous quelles hypothèses. Chaque formule correspond à une fonction
identifiable de `src/core`, indiquée en marge.

**Convention d'unités.** Tous les montants sont en euros, tous les taux en
pourcentage annuel, toutes les durées en années entières.

---

## 1. Unité de décision

Une **alternative** est un couple (bien, montage de financement), non un bien
seul. Le même appartement financé à 40 % ou à 50 % d'apport donne deux
alternatives, parce que le financement modifie à la fois la rentabilité perçue
par l'investisseur et la faisabilité bancaire de l'opération.

Les hypothèses partagées — horizon de détention, taux d'actualisation, seuil
DSCR, régime fiscal — sont attachées au **projet**, non à l'alternative.
Comparer deux VAN calculées sur des horizons ou des coûts d'opportunité
différents reviendrait à comparer deux échelles.

---

## 2. Coût d'acquisition

> `analyze.mjs`

```
Coût total = Prix × (1 + f)
Emprunt    = max(Coût total − Apport, 0)
```

`f` désigne les frais d'acquisition — notaire, agence, garantie — exprimés en
pourcentage du prix. Ils sont supposés financés par l'apport, ce qui correspond
à la pratique bancaire courante.

Ces frais représentent en pratique 7 à 8 % du prix dans l'ancien. Les omettre ne
décale pas seulement la VAN : cela déplace le montant emprunté, donc l'échéance,
donc le DSCR — c'est-à-dire le verdict bancaire lui-même.

---

## 3. Échéancier de prêt

> `loan.mjs`

Prêt à annuités constantes : l'investisseur rembourse la même somme chaque
année, mais la répartition entre intérêts et capital évolue.

```
Échéance = C × t / [1 − (1 + t)⁻ⁿ]
```

`C` = montant emprunté, `t` = taux annuel, `n` = durée du prêt.

L'échéancier est décomposé année par année :

```
Intérêts(k)   = Capital restant dû(k−1) × t
Amortissement = Échéance − Intérêts(k)
```

Cette décomposition fournit trois informations dont le reste du modèle dépend :
la part d'intérêts de chaque année (déductible au régime réel), le service de la
dette annuel (dénominateur du DSCR), et le capital restant dû à toute date
(nécessaire en cas de revente avant la fin du prêt).

Quand la durée de détention excède la durée du prêt, les années suivantes portent
un service de dette nul et aucun DSCR.

---

## 4. Exploitation : loyers et charges indexés séparément

> `analyze.mjs`

```
Loyer encaissé(k) = Loyer brut × (1 + gL)^(k−1) × (1 − v)
Charges(k)        = Charges × (1 + gC)^(k−1)
Loyer net(k)      = Loyer encaissé(k) − Charges(k)
```

`gL` et `gC` sont les croissances annuelles respectives du loyer et des charges,
`v` le taux de vacance locative.

N'indexer que les loyers rendrait le modèle optimiste par construction :
copropriété, entretien et taxe foncière suivent eux aussi l'inflation, souvent
plus vite que l'indexation réglementée des loyers.

C'est ce **loyer net** — et non le loyer brut — qui alimente aussi bien les flux
de l'investisseur que le calcul du DSCR.

---

## 5. Fiscalité

> `tax.mjs`

Périmètre : location nue détenue par un particulier résident, hors résidence
principale. Les régimes LMNP et LMP, ainsi que les dispositifs de
défiscalisation, ne sont pas modélisés.

### Revenus fonciers

**Micro-foncier** — abattement forfaitaire de 30 % sur le loyer brut :

```
Base = Loyer encaissé × 0,70
Impôt = Base × (TMI + PS)
```

Le régime n'est ouvert que sous 15 000 € de recettes annuelles ; au-delà, l'outil
le signale.

**Régime réel** — charges et intérêts d'emprunt sont déductibles :

```
Base = Loyer encaissé − Charges − Intérêts − Déficit reporté
```

Une base négative ne produit pas d'économie immédiate : elle constitue un déficit
reporté sur les revenus fonciers des années suivantes. Cette mécanique compte
lourdement dans les premières années d'un montage à fort levier, où les intérêts
dominent l'échéance.

### Plus-value de cession

Optionnelle. Abattements pour durée de détention :

| | Impôt sur le revenu (19 %) | Prélèvements sociaux (17,2 %) |
|---|---|---|
| Années 6 à 21 | 6 %/an | 1,65 %/an |
| Année 22 | 4 % → exonération | 1,60 % |
| Années 23 à 30 | — | 9 %/an → exonération |

Une moins-value n'est pas imposée et n'ouvre aucun droit à imputation.

---

## 6. Rentabilité de l'investisseur

> `cashflow.mjs`

VAN et TRI portent sur les flux réellement perçus par l'investisseur au titre de
ses **fonds propres**, non sur la performance brute du bien.

| Date | Flux |
|------|------|
| Année 0 | − Apport personnel |
| Années 1 à n | Loyer net − Échéance − Impôt foncier |
| Année n (ajout) | + Valeur de revente − Capital restant dû − Impôt sur la plus-value |

```
VAN = Σ CF(k) / (1 + a)^k
TRI : le taux r tel que Σ CF(k) / (1 + r)^k = 0
```

`a` est le taux d'actualisation, reflet du coût d'opportunité de l'investisseur.

Le TRI n'ayant pas de forme close, il est résolu par balayage puis dichotomie sur
l'intervalle [−99 %, +300 %]. Le résultat porte un **statut explicite** :

- *unique* — cas normal, une seule racine ;
- *multiple* — la série change plusieurs fois de signe, plusieurs TRI existent et
  l'indicateur perd son sens ; l'outil le dit et renvoie à la VAN ;
- *aucune* — apport très faible, levier extrême : aucune racine sur
  l'intervalle. L'outil affiche « Indéterminé » plutôt qu'une valeur trompeuse.

Un TRI qui n'existe pas est une information à afficher, pas une erreur à masquer.

### Effet de levier

Emprunter plutôt que payer comptant augmente le rendement des capitaux engagés,
tant que le coût de la dette reste inférieur au rendement du bien. L'outil calcule
systématiquement un TRI de référence correspondant au même bien acheté comptant,
toutes autres hypothèses inchangées :

```
Effet de levier = TRI(avec emprunt) − TRI(comptant)
```

---

## 7. Faisabilité bancaire

> `analyze.mjs`

```
DSCR(k) = Loyer net(k) / Échéance(k)
```

Le DSCR est l'indicateur du prêteur, non de l'investisseur : il conditionne
l'accès au financement, indépendamment de la rentabilité du projet. Il est
calculé **avant impôt**, conformément à la lecture bancaire.

La règle est binaire et porte sur toute la durée du prêt : une alternative n'est
finançable que si son DSCR reste au-dessus du seuil — 1,20 par défaut — **chaque
année sans exception**. Une seule année sous le seuil suffit à l'écarter. Ni
moyenne, ni contrôle limité à la première année : une banque examine l'échéancier
projeté dans son intégralité.

Comme le loyer et les charges progressent à des rythmes différents, l'année la
plus fragile n'est pas nécessairement la première.

### Marge de résistance

Plutôt que de demander à l'utilisateur d'imaginer un scénario de choc — quelle
année, quelle ampleur, deux paramètres qu'il ne peut pas connaître — l'outil
renverse la question : combien de loyer une alternative peut-elle perdre, dans
son année la plus fragile, avant de rompre le seuil ?

```
Marge = 1 − (Seuil × Échéance) / Loyer net,   évaluée à l'année de DSCR minimal
```

Le terme retranché représente la part du loyer déjà mobilisée par l'exigence
bancaire ; la marge est ce qui reste. Une marge de 20 % équivaut à environ
2,4 mois de loyer perdus sans casser le financement.

---

## 8. Décision multicritère

> `scoring.mjs`

Les alternatives non finançables sont d'abord écartées. Les autres sont comparées
sur quatre critères, tous à maximiser :

| Critère | Rôle |
|---------|------|
| VAN | Création de valeur actualisée pour l'investisseur |
| TRI | Rendement annualisé des fonds propres engagés |
| Rendement locatif net | Performance courante, indépendante du financement |
| Marge de sécurité DSCR | Confort vis-à-vis du seuil, parmi les alternatives finançables |

Cette méthode a été préférée à une optimisation combinatoire ou à un arbre de
décision : le nombre d'alternatives est limité et connu à l'avance, les critères
sont hétérogènes en unité, et une méthode transparente se justifie et se fait
varier plus facilement devant un comité.

```
normalisé = (valeur − min) / (max − min)
Score     = Σ poids(k) × normalisé(k)
```

Les bornes sont calculées sur les seules alternatives finançables : une
alternative écartée ne doit pas étirer l'échelle des autres.

Trois cas particuliers sont traités explicitement :

- **achat comptant** — la marge de sécurité vaut 1 : en l'absence de dette,
  aucun engagement bancaire ne peut être rompu ;
- **TRI indéterminé** — le critère vaut 0 : l'indicateur ne peut pas soutenir la
  comparaison ;
- **critère constant sur tout le lot** — normalisé à 1 : il ne discrimine rien,
  il ne doit donc pénaliser personne.

Les poids fonctionnent comme un budget de cent points : augmenter un critère
réduit les trois autres au prorata, de sorte que la position d'un curseur
corresponde toujours exactement au pourcentage affiché. L'arrondi au plus fort
reste garantit que les pourcentages affichés totalisent exactement 100.

Les priorités sont fixées **avant** l'affichage du classement. Si l'utilisateur
voyait d'abord quelle alternative a la meilleure VAN, il réglerait ses poids pour
la faire gagner : le score ne servirait plus à décider, mais à justifier après
coup.

---

## 9. Robustesse

> `robustness.mjs`

Un score pondéré ne vaut que ce que valent ses poids, et personne ne connaît les
siens à cinq points près. Trois analyses répondent à cette fragilité.

### Stabilité aux poids

Deux mille pondérations sont tirées autour du réglage retenu (±25 points par
critère, renormalisées à 100), et le classement est recalculé pour chacune.
L'outil rapporte la fréquence à laquelle chaque alternative reste première.

Le générateur pseudo-aléatoire est **déterministe** : deux exécutions identiques
produisent le même chiffre. Un résultat qui change d'un affichage à l'autre ne se
présente pas devant un comité.

Lecture retenue : au-delà de 80 % de victoires, le classement est robuste ;
entre 55 et 80 %, il est nuancé ; en deçà, il dépend surtout des poids et
l'arbitrage doit se faire sur d'autres bases.

### Points de bascule

Pour chaque critère, l'outil balaie le poids de 0 à 100 % — les autres étant
redistribués au prorata — et repère la valeur à partir de laquelle le vainqueur
change. Cela transforme une intuition en phrase vérifiable : « en deçà de 6 % de
poids sur le TRI, B passe devant ».

### Scénarios de choc

Cinq hypothèses dégradées, une à la fois — loyer −10 %, vacance +5 points, taux
+1 point, charges +20 %, revente −10 % — chacune rejouée sur l'analyse complète.
Les chocs ne sont pas cumulés : une sensibilité lisible vaut mieux qu'un scénario
composite dont on ne sait plus quelle variable a produit l'effet.

---

## 10. Exemple chiffré

Trois montages du même bien. Prix 200 k€, frais d'acquisition 7,5 %, loyer brut
1 100 €/mois, charges 1 800 €/an, croissances de 1,5 % (loyer) et 2,5 %
(charges), revente estimée à 230 k€ à dix ans, actualisation 4 %, seuil DSCR
1,20, fiscalité non modélisée.

| Indicateur | A — apport 80 k€, 4,20 %, 20 ans | B — apport 100 k€, 4,00 %, 20 ans | C — apport 100 k€, 4,15 %, 25 ans |
|---|---|---|---|
| Montant emprunté | 135,0 k€ | 115,0 k€ | 115,0 k€ |
| Échéance annuelle | 10,11 k€ | 8,46 k€ | 7,48 k€ |
| DSCR minimum | **1,128** | 1,347 | 1,524 |
| VAN | 36,34 k€ | **38,19 k€** | 36,95 k€ |
| TRI | 8,24 % | 7,82 % | 7,85 % |
| Effet de levier | +2,12 pts | +1,70 pts | +1,73 pts |
| Marge de résistance | −6,4 % | 10,9 % (1,3 mois) | 21,3 % (2,6 mois) |
| Capital restant dû à 10 ans | 81,2 k€ | 68,6 k€ | 82,3 k€ |
| Score (profil équilibré) | — écartée | 0,450 | **0,700** |

**A est écartée** avant tout scoring : son DSCR tombe à 1,128, sous le seuil. Elle
affiche pourtant le meilleur TRI et le meilleur effet de levier — c'est
exactement le montage qu'un classement sans filtre de faisabilité aurait
recommandé, et qu'aucune banque n'aurait financé.

Ce basculement tient aux frais d'acquisition : sans eux, A passait le seuil avec
un DSCR de 1,27. Les 15 000 € de frais augmentent l'emprunt, donc l'échéance,
donc le verdict bancaire.

**Entre B et C**, l'arbitrage est réel. B dégage la meilleure VAN — l'économie
d'intérêts sur vingt ans compense le taux légèrement inférieur. C allonge le prêt
à vingt-cinq ans : l'échéance baisse d'un millier d'euros par an, la marge de
résistance double, mais le capital restant dû au terme est plus élevé de 13,7 k€.
Avec le profil équilibré, C l'emporte à 0,700 contre 0,450.

**Robustesse.** C reste première dans 84 % des pondérations voisines. Le
classement bascule si le poids de la VAN dépasse 44 %, ou si celui du TRI descend
sous 6 % — deux réglages atteignables, que l'analyse rend explicites plutôt que
de les laisser en suspens.

---

## 11. Limites

- Prêt à annuités constantes uniquement : ni taux variable, ni différé
  d'amortissement, ni palier.
- Flux annuels, pas mensuels : l'échéance et le loyer sont supposés perçus en
  bloc en fin d'année.
- La somme pondérée est totalement compensatoire : une marge de sécurité faible
  peut être rachetée par une VAN élevée. Des méthodes de surclassement comme
  ELECTRE ou PROMETHEE limiteraient cette compensation.
- La normalisation min-max dépend du lot d'alternatives présentes : en ajouter ou
  en retirer une déplace les bornes et peut inverser un classement — le
  *rank reversal*. L'analyse de stabilité mesure ce risque, elle ne le supprime
  pas.
- Les chocs sont testés isolément. Une simulation probabiliste corrélée, de type
  Monte-Carlo sur les hypothèses de marché plutôt que sur les poids, constitue
  l'extension naturelle du modèle.
- Le seuil DSCR et les taux de croissance sont des hypothèses de travail
  ajustables, non des règles universelles : les exigences varient selon les
  banques et les marchés.
