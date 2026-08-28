# Note de reprise — de la v1 à la v3

Ce document liste ce qui a changé par rapport à la première version, et
pourquoi. Il sert de trace pour le rapport et la soutenance : chaque décision
peut être défendue.

---

## Ce qui a été conservé

Le modèle financier d'origine était juste. Sa formulation — annuités constantes,
VAN sur les flux de fonds propres, DSCR sur toute la durée du prêt, marge de
résistance calculée plutôt que devinée, scoring pondéré avec filtre de
faisabilité en amont — n'a pas été remise en cause. Ce sont les bonnes décisions
de modélisation, et elles structurent toujours l'outil.

**Vérification de continuité.** L'exemple chiffré de la fiche descriptive
(alternatives A et B, sans frais ni fiscalité) est rejoué par le nouveau moteur
dans `test/engine.test.mjs` : échéance 8,99 k€, DSCR 1,269, VAN 51,54 k€, TRI
9,889 %, capital restant dû 72,2 k€, marge 5,4 %. Tous les chiffres déjà publiés
restent valides.

---

## Corrections de modèle

### Frais d'acquisition (nouveau)

Les frais de notaire, d'agence et de garantie — 7 à 8 % du prix dans l'ancien —
n'étaient pas modélisés. Ce n'est pas un détail de précision : ils augmentent le
montant emprunté, donc l'échéance, donc le DSCR. Dans l'exemple de
`docs/modele.md`, leur prise en compte fait basculer l'alternative A de
finançable (DSCR 1,27) à refusée (DSCR 1,13).

### Fiscalité (nouveau)

Absente de la v1, elle est désormais optionnelle et explicite : micro-foncier,
régime réel avec déficit foncier reportable, et imposition de la plus-value de
cession avec les abattements pour durée de détention. Sur l'exemple de référence,
passer d'un modèle avant impôt au régime réel avec plus-value ramène la VAN de
38 k€ à 4 k€ — un ordre de grandeur qu'un investisseur ne peut pas ignorer.

### Vacance locative (nouveau)

Un taux de vacance par alternative, appliqué au loyer encaissé. Sans lui, le DSCR
suppose implicitement un bien loué douze mois sur douze pendant toute la durée du
prêt.

### Horizon et actualisation remontés au niveau du projet (correction)

En v1, la durée de détention et le taux d'actualisation étaient saisis par
alternative. Deux VAN calculées sur des horizons ou des coûts d'opportunité
différents ne sont pas comparables — le classement pouvait donc comparer des
échelles, pas des projets. Ces deux hypothèses sont maintenant communes.

### Unités homogènes (correction)

La v1 mélangeait les k€ (prix, charges, revente) et les euros (loyer mensuel)
dans le même objet, avec des conversions dispersées dans le moteur. Tout est en
euros dans le calcul ; la conversion en k€ appartient exclusivement au formatage.

### Statut du TRI (correction)

La v1 renvoyait une valeur de bord accompagnée d'un drapeau « extrême » quand la
dichotomie échouait. Le nouveau moteur distingue trois cas — racine unique,
racines multiples, aucune racine — et l'interface les explique. Un TRI multiple
est signalé, ce que la v1 ne détectait pas du tout.

---

## Ce qui manquait au projet, pas au modèle

### Analyse de robustesse (livrable de la semaine 5)

La feuille de route prévoyait « faire varier les poids pour vérifier la stabilité
du classement obtenu ». Ce n'avait pas été fait. Trois analyses le font
maintenant : stabilité sur 2 000 pondérations tirées, points de bascule critère
par critère, et cinq scénarios de choc rejoués intégralement. Le générateur
aléatoire est déterministe, pour que le chiffre présenté soit reproductible.

### Tests (nouveau)

La v1 n'en avait aucun. Un outil d'aide à la décision financière sans test
unitaire n'est pas remettable : rien ne garantit qu'une modification ultérieure
ne casse pas un calcul. Il y a maintenant 30 tests sur le moteur — incluant la
non-régression sur l'exemple publié, les cas limites (taux nul, achat comptant,
prêt plus court que la détention, moins-value) et les invariants (les poids
somment à 100, les fréquences de victoire somment à 1) — plus un test de bout en
bout qui pilote le fichier livré dans un DOM simulé.

### Validation des saisies (nouveau)

La v1 acceptait n'importe quelle valeur. Deux niveaux existent désormais :
l'erreur bloquante retire l'alternative du lot, l'avertissement la laisse passer
en signalant l'hypothèse douteuse. Chaque message dit quoi faire, pas seulement
que la valeur est invalide.

### Écran de connexion retiré

La v1 affichait un formulaire avec `demo1234` en clair dans le code source. Une
authentification qui ne protège rien donne une fausse impression de sécurité —
et c'est le premier détail qu'un lecteur extérieur remarque. L'outil est
local-first : les données restent sur le poste, il n'y a rien à authentifier.

### Persistance et export (nouveau)

Les projets survivent à la fermeture du navigateur (`localStorage`, avec repli en
mémoire quand le navigateur refuse). Le classement et la projection détaillée
s'exportent en CSV, et la synthèse s'imprime avec une feuille de style dédiée.

---

## Architecture

La v1 tenait en un fichier HTML de 90 Ko : moteur, état et rendu dans la même
portée, sans test, sans typage, sans build. La contrainte de livraison — un
fichier ouvrable par double-clic — était la bonne, mais elle ne devait pas
dicter l'organisation du code source.

Le projet repose désormais sur une pile standard, choisie pour être reprise par
quelqu'un d'autre sans explication :

| Choix | Raison |
|---|---|
| **TypeScript** strict | Le domaine est numérique et ses unités se confondent : un taux, un montant et un ratio sont tous des `number`. Les types portent la convention et rendent visibles les valeurs absentes — `irr: number \| null` oblige chaque appelant à traiter le cas où le TRI n'existe pas. |
| **React** | Le rendu doit refléter la saisie en continu. La réconciliation gère le focus et le curseur du champ édité, ce que la version précédente traitait à la main. |
| **Zustand** | Un store global sans passe-plat. Le middleware `persist` porte la sauvegarde locale, la version du schéma et le repli quand le navigateur refuse le stockage. |
| **Recharts** | Axes, graduations, info-bulles et redimensionnement, testés bien au-delà d'un tracé SVG maison — c'est précisément là que la version précédente produisait des graduations dupliquées. |
| **Vite** | Rechargement à chaud en développement, fichier unique en livraison. |
| **Vitest** + **Playwright** | Le moteur testé en isolation, le fichier livré piloté dans un vrai navigateur. |

Le fichier livré pèse environ 660 Ko contre 136 Ko sans framework. React et
Recharts expliquent l'essentiel de l'écart : c'est le prix assumé de la
solidité, pour un fichier local chargé une fois et sans requête réseau.

Le détail est dans [`architecture.md`](architecture.md).

## Interface

La direction visuelle a changé : d'une esthétique de brochure — violet profond,
titres en serif, dégradés, silhouette de ville en pied de page — vers celle d'un
outil d'analyse patrimoniale. Fond très clair légèrement bleuté, cartes blanches
détachées par une ombre douce, un indigo profond pour ce qui est retenu, un rouge
sourd pour ce qui est refusé par la banque. Deux règles typographiques tenues
partout : les libellés sont écrits comme des mots — pas en capitales espacées —
et la chasse fixe est réservée aux chiffres qui doivent s'aligner en colonne.

L'élément central de l'étape Résultat est le **plan de décision** : chaque
alternative placée selon sa rentabilité (TRI) et sa résistance, avec le seuil
bancaire tracé comme un mur hachuré. Les alternatives écartées restent visibles
de l'autre côté — savoir pourquoi une piste tombe vaut mieux que de la voir
disparaître.

L'étape Résultat reste verrouillée tant que les priorités ne sont pas validées.
La règle existait en v1 dans l'ordre des écrans ; elle est maintenant appliquée.

Le projet de démonstration ouvre sur trois montages du même bien, choisis pour
que le premier écran montre un arbitrage réel plutôt qu'une comparaison sans
enjeu : l'un est écarté par la banque malgré le meilleur TRI, les deux autres
s'échangent la première place selon les poids retenus.

### Deux bugs trouvés par l'outillage de test

**La virgule décimale effacée.** Le rendu intégral de la v2 réécrivait le champ
en cours d'édition à chaque frappe. Taper « 4,2 » dans un champ de taux voyait
la virgule disparaître dès sa saisie — la valeur interne valant déjà 4 — et
produisait « 42 ». React conserve le nœud DOM, ce qui règle le focus et le
curseur ; le champ garde en plus un brouillon local tant qu'il est actif, et la
mise en forme ne reprend qu'à la sortie. Un test tape « 4,25 » caractère par
caractère dans un vrai navigateur.

**Les modifications perdues.** Au portage React, la lecture du projet courant
appliquait un repli sur le premier de la liste quand aucun projet n'était
sélectionné, mais pas l'écriture. Résultat : au premier démarrage, toutes les
saisies portaient sur aucun projet et disparaissaient sans le moindre message
d'erreur. Le parcours de bout en bout l'a détecté immédiatement — les curseurs
de poids restaient figés sur leurs valeurs par défaut. Lecture et écriture
passent désormais par la même fonction de résolution.

### Revue visuelle outillée

`npm run screenshots` pilote un navigateur sans interface, parcourt les cinq
étapes, vérifie treize points critiques et capture chaque écran. Une suite de
tests dit qu'un bouton existe ; elle ne dit pas qu'il est lisible. Cinq défauts
ont été trouvés ainsi : des graduations d'axe dupliquées par arrondi, des
étiquettes superposées sur le plan de décision, des points de bascule dégénérés
annonçant qu'un critère porté à 100 % de poids change le classement, une
étiquette de seuil tronquée au bord du graphique, et des libellés de points
coupés mot à mot par le rendu par défaut de Recharts.
