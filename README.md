# ImmoDécision

**Outil d'aide à la décision pour l'investissement et la levée de fonds immobiliers.**

ImmoDécision aide un investisseur à choisir, parmi plusieurs biens et plusieurs
montages de financement, celui qui offre le meilleur compromis entre rentabilité
et risque — en écartant automatiquement les montages qu'une banque refuserait de
financer, et en disant à quel point cette réponse tient si l'on change d'avis
sur les priorités ou si le marché se retourne.

Le principe central : **une alternative n'est pas un bien, c'est un couple
(bien, montage de financement)**. Un même appartement financé avec 40 % ou 50 %
d'apport constitue deux alternatives distinctes, parce que le financement change
à la fois la rentabilité perçue par l'investisseur et la faisabilité bancaire de
l'opération.

---

## Démarrer

### Utiliser l'outil

Ouvrir `dist/index.html` dans un navigateur. Aucun serveur, aucune installation,
aucune requête réseau. Les projets sont conservés localement sur le poste.

### Développer

```bash
npm install
npm run dev          # serveur de développement, rechargement à chaud
npm test             # 30 tests du moteur de calcul (Vitest)
npm run typecheck    # TypeScript en mode strict
npm run build        # typage + bundle → dist/index.html
npm run screenshots  # parcours de bout en bout + captures des cinq écrans
npm run check        # typage, tests et build enchaînés
```

---

## Pile technique

| Choix | Raison |
|---|---|
| **TypeScript** strict | Le domaine est numérique et les unités s'y confondent facilement. Les types rendent impossible de passer un taux là où un montant est attendu, ou d'oublier qu'un TRI peut être `null`. |
| **React** | Le rendu doit refléter la saisie en continu sur cinq écrans. La réconciliation gère le focus et le curseur du champ édité, ce qu'une couche de rendu maison imposait de traiter à la main. |
| **Zustand** | Un store global sans passe-plat ni contexte imbriqué. Le middleware `persist` porte la sauvegarde locale, la version du schéma et le repli quand le navigateur refuse le stockage. |
| **Recharts** | Axes, graduations, info-bulles et redimensionnement, testés bien au-delà de ce qu'un tracé SVG maison peut garantir. |
| **Vite** + `vite-plugin-singlefile` | Développement avec rechargement à chaud, livraison en un fichier HTML autonome. |
| **Vitest** + **Playwright** | Le moteur est testé en isolation ; le fichier livré est piloté dans un vrai navigateur. |

---

## Parcours

| # | Étape | Rôle |
|---|-------|------|
| 1 | Hypothèses | Horizon, taux d'actualisation, seuil DSCR, régime fiscal |
| 2 | Alternatives | Saisie des couples bien × financement, vérifiée en continu |
| 3 | Priorités | Pondération des quatre critères |
| 4 | Calculs | Projection année par année de chaque alternative |
| 5 | Résultat | Classement, plan de décision, robustesse, scénarios de choc |

Les priorités sont fixées **avant** l'affichage du classement. Si l'utilisateur
voyait d'abord quelle alternative a la meilleure VAN, il réglerait ses poids pour
la faire gagner : le score ne servirait plus à décider, mais à justifier après
coup. L'étape 5 reste verrouillée tant que l'étape 3 n'est pas validée.

---

## Indicateurs

**VAN et TRI** portent sur les flux réellement perçus par l'investisseur au titre
de ses fonds propres, pas sur la performance brute du bien. Le TRI est résolu
numériquement ; quand aucune racine n'existe sur l'intervalle exploré, ou qu'il
en existe plusieurs, l'outil le dit et renvoie à la VAN plutôt que d'afficher un
chiffre trompeur.

**Effet de levier** — écart entre le TRI avec emprunt et celui du même bien
acheté comptant, toutes autres hypothèses inchangées.

**DSCR** — capacité du bien à couvrir seul l'échéance de prêt. Une alternative
n'est finançable que si son DSCR reste au-dessus du seuil bancaire **chaque
année, sans exception**. Une banque examine l'échéancier projeté en entier.

**Marge de résistance** — combien de loyer une alternative peut perdre, dans son
année la plus fragile, avant de rompre le seuil. Plutôt que de demander à
l'utilisateur d'imaginer un choc, l'outil renverse la question.

**Score multicritère pondéré** — VAN, TRI, rendement locatif net et marge de
sécurité DSCR, normalisés puis agrégés selon des poids réglables (budget de cent
points). Trois profils de départ : prudent, équilibré, offensif.

**Robustesse** — deux mille pondérations tirées autour du réglage retenu, points
de bascule critère par critère, et cinq chocs d'hypothèse rejoués intégralement.

---

## Architecture

```
src/core/         moteur de calcul — fonctions pures, ni DOM ni état
src/store/        état Zustand et sélecteurs dérivés
src/components/   interface React (ui, charts, steps)
src/lib/          formatage, export CSV, jetons de couleur
test/             30 tests du moteur
e2e/              parcours complet dans un navigateur réel
```

Le moteur ne dépend ni de React ni du navigateur, et s'utilise seul :

```ts
import { analyze, rank, defaultSettings } from './src/core';
```

Le détail des choix est dans [`docs/architecture.md`](docs/architecture.md).

---

## Hypothèses et limites

- Prêt à annuités constantes uniquement : ni taux variable, ni différé, ni palier.
- Flux annuels, pas mensuels.
- Fiscalité : location nue détenue par un particulier. Les régimes LMNP et LMP,
  ainsi que les dispositifs de défiscalisation, ne sont pas modélisés.
- La somme pondérée est compensatoire : une marge de sécurité faible peut être
  rachetée par une VAN élevée. Des méthodes de surclassement (ELECTRE,
  PROMETHEE) limiteraient cette compensation.
- La normalisation min-max dépend du lot d'alternatives présentes : en ajouter ou
  en retirer une déplace les bornes et peut inverser un classement — le
  *rank reversal*. L'analyse de robustesse mesure ce risque sans le supprimer.
- Le seuil DSCR et les taux de croissance sont des hypothèses de travail
  ajustables, pas des règles universelles.

---

## Auteurs

- [ShemTychyque](https://github.com/ShemTychyque) — SAGBO Shem
- [jaguaro-gawin](https://github.com/jaguaro-gawin) — OGAWIN Salami
