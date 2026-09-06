# PallyPowerTW — Audit complet & plan d'amélioration

> Document de travail (le « prompt ») pour guider les corrections et améliorations de l'addon.
> Cible : WoW 1.12 / Turtle WoW. Fichier principal : `PallyPower.lua` (~3750 lignes).
> Version TOC : 1.40 — Auteur d'origine : ivanovlk.

---

## 0. État actuel (important)

**La logique de cast est revenue à celle de l'addon d'origine (ivanovlk).** Après plusieurs tentatives
d'ajouter un override de bénédiction *individuelle* au clic droit (poser une béné différente de celle
de classe sur un membre précis), on a bute sur des pièges d'API WoW 1.12 trop instables pour rester en
production (`CastSpell` qui annule le ciblage s'il recastait le même sort ; `SpellCanTargetUnit` qui
refuse un downgrade de rang ; conflits de priorité joueur/alliés). Plutôt que d'empiler des correctifs,
**les deux fonctions de cast (`PallyPowerBuffButton_OnClick`, `PallyPower_AutoBless`) ont été restaurées
à l'identique de la version de base** (commit `a994904`, avant tout travail sur l'individuel).

Ce qui est **conservé** par-dessus cette base :
- Le realm **N'Zoth** dans `TURTLE_REALMS` (durées de béné correctes).
- Tout le **durcissement anti-crash** (voir §2 et §6) — indépendant de la logique de cast.
- `nhave_class` (couleur/timer de la barre de buff ne se réinitialisent plus à tort).

**Réintroduction propre des bénédictions individuelles** (après validation du socle) :
le cast individuel est maintenant une **passe séparée** — `PallyPower_CastIndividualBlessing(classID)`
(~ligne 3116) — appelée en tête du clic droit / Hotkey1, AVANT que le sort de classe soit mis sur le
curseur. La logique de classe d'origine reste **intouchée** (elle ne s'exécute que si la passe
individuelle n'a rien casté). Cette architecture évite les deux pièges 1.12 **par construction** :
- aucun sort n'est en attente quand la passe individuelle caste → pas de toggle-off possible ;
- elle ne cible que des membres qui n'ont **pas** la béné individuelle → pas de refus de downgrade
  (un membre avec la GB de classe peut recevoir une individuelle DIFFÉRENTE sans problème).

Comportement : un cast par clic. Les clics posent d'abord les individuelles manquantes (une par clic),
puis retombent sur la béné de classe classique. Anti double-cast de 5 s par personne (le temps que le
scan de buffs voie le nouveau buff). Cas de test à valider en jeu :
soi-même sans GB ✱ soi-même avec GB ✱ allié sans GB ✱ allié avec GB ✱ deux individuelles différentes
sur deux membres (ex. GB Salut sur la classe + Might sur soi + Wisdom sur un allié → 2 clics).

**Sémantique « remplace, pas cumule » (retour d'ilivans, l'auteur upstream, 14/07/2026) :**
une assignation individuelle REMPLACE la béné de classe pour ce membre — elle ne s'y ajoute pas.
Bug rapporté : classe + individuelle assignées sur soi, 2 clics droits → les DEUX bénés étaient
posées (l'individuelle par la pré-passe, puis la classe par la boucle car la gate ne filtre pas sur
`btn.need`). Fix : la boucle de classe (clic droit / Hotkey1 uniquement) **saute** tout membre ayant
une individuelle castable (`skipIndividual` : `GetNormalBlessings ~= -1 et AllPallys[joueur][id]`
non nil). Cohérent avec le `hasBuffs` de l'UI qui ne compte ces membres couverts que par leur
individuelle — le bouton vert après la pose de l'individuelle est donc correct. Le clic gauche
(Grande Béné) ne saute personne : la GB couvre toute la classe par nature. Si l'individuelle n'est
pas castable (talent/rang manquant), le membre retombe sur la béné de classe (pas d'angle mort).

---

## 1. Vue d'ensemble

PallyPowerTW coordonne les **bénédictions de paladin** (et auras/sceaux) dans un groupe/raid :
assignations par classe, synchronisation entre paladins via messages addon, scan des buffs présents,
et cast en un clic (Grande Bénédiction 30 min = clic gauche, bénédiction 10 min = clic droit).

### Systèmes principaux
| Système | Fonctions clés | Rôle |
|---------|----------------|------|
| Assignations | `*_PerformCycle*`, `GetNormalBlessings`, `SetNormalBlessings` | Choix des bénés par classe / par joueur |
| Sync réseau | `PallyPower_SendSelf`, `PallyPower_ParseMessage`, `*_TacticaParseMessage` | Partage état entre paladins (+ intégration Tactica) |
| Scan buffs | `PallyPower_ScanRaid`, `ScanOneUnit`, `RebuildRoster`, `PruneCurrentBuffs` | État des buffs présents |
| Scan sorts | `PallyPower_ScanSpells`, `PallyPowerManaCost.lua` | Rangs/talents dispo du joueur |
| Cast | `PallyPowerBuffButton_OnClick`, `PallyPower_AutoBless`, `PallyPower_CastSeal` | Lancement des sorts (béné de classe uniquement) |
| Presets | `MinimapButton.lua` (`PallyPower_SaveSet`, `PallyPower_SwapSet`...) | Sauvegarde/chargement de jeux d'assignations |
| UI | `PallyPower_UpdateUI`, `PallyPowerGrid_Update`, `PallyPower_UpdateLayout`, `PallyPower.xml` | Affichage |
| Boucle | `PallyPower_OnUpdate`, `PallyPower_OnEvent` | Timers, debounce UI, événements |

### Fichiers du projet
| Fichier | Rôle |
|---------|------|
| `PallyPower.lua` | Logique principale (~3750 lignes) |
| `PallyPower.xml` | Layout UI — boutons fins qui appellent le Lua, pas de logique embarquée |
| `MinimapButton.lua` / `.xml` | Bouton minimap + presets d'assignations |
| `PallyPowerManaCost.lua` | Table de coût en mana par rang (voir §7, non branché) |
| `Locale/localization-*.lua` | Traductions (enUS, deDE, esES) |
| `Bindings.xml` | Raccourcis clavier (toggle, autobless, seal) |

### Protocole réseau (préfixe `PP_PREFIX`, canal PARTY/RAID)
- `SELF`/`ASELF`/`SSELF` : rangs+talents + assignations du paladin émetteur (bénés/auras/sceaux).
- `ASSIGN`/`AASSIGN`/`SASSIGN`/`MASSIGN`/`NASSIGN` : assignation (classe / aura / sceau / toutes / individuelle).
- `SYMCOUNT`, `COOLDOWNS`, `FREEASSIGN`, `TANK`/`CLTNK`, `HEALER`/`CLHLR`, `CLEAR`, `VERSION`, `REQ`.
- Intégration **Tactica** (préfixe `TACTICA_PREFIX`) : synchro tank/heal marks avec cet addon tiers.

---

## 2. Bugs & risques (par sévérité) — état après durcissement

### 🔴 Critiques — CORRIGÉS (voir §6 pour le détail de chaque fix)
1. `COOLDOWNS` indexait `AllPallys[sender]` sans garde → crash. **Corrigé.**
2. `VERSION` comparait un `msgVer` potentiellement nil → crash + tri lexicographique faux. **Corrigé.**
3. `SELF`/`ASELF`/`SSELF` : `tmp + 0` sur caractère non validé → crash. **Corrigé.**
4. `talent + 0` dans l'UI (grille des paladins) → crash à chaque frame si talent non numérique. **Corrigé.**
5. **Tactica** : `n ~= ""` est vrai même si `n` est `nil` (un message tronqué comme `"S:T"` sans 3e
   segment donne `n = nil`), et `PallyPower_Tanks[nil] = true` lève une erreur Lua ("table index is
   nil"). Trois occurrences : `PallyPower.lua:~1969, ~2003, ~2015`. **Corrigé** (`n and n ~= ""`).
6. `PallyPower_ScanSpells` : la branche Grande Bénédiction (`RegularBlessings == false`, mode par
   défaut) écrit `RankInfo[id]["id"] = i` en supposant que `RankInfo[id]` existe déjà (rempli par le
   scan de la béné "petite" plus tôt dans la même boucle). Si l'ordre du grimoire place un jour la
   Grande Béné avant la petite pour une classe donnée, `nil` indexé → crash. **Corrigé** (garde
   `elseif RankInfo[id] then`, dégradation gracieuse au lieu du crash).

### 🟠 Moyens — CORRIGÉS
7. **`shift` global instable dans `PallyPower_PerformCycle`/`PerformCycleBackwards`**
   (`PallyPower.lua:~2612, ~2700`) : contrairement à leurs 4 fonctions sœurs (Aura/Seal, qui font
   `local shift = IsShiftKeyDown()` sans condition), les cycles de bénédiction ne lisaient l'état de
   Shift QUE si `skipempty == false`. Si un appel précédent (ex: clic normal avec Shift enfoncé)
   laissait la variable globale `shift = true`, un appel suivant sans rapport (ex: molette sur un
   autre bouton, `skipempty ~= false`) héritait de cette valeur périmée et déclenchait silencieusement
   la branche `MASSIGN` — écrasant **toutes** les 10 assignations de classe du joueur ciblé alors que
   personne n'appuyait sur Shift. **Corrigé** : `local shift = IsShiftKeyDown()` inconditionnel,
   aligné sur le pattern des 4 autres fonctions.
8. **Blocage de cycle au simple-clic (`PerformCycle` bloqué à -1)** : Si `NeedsBuff` renvoie false pour
   tous les sorts restants (doublons sur un autre paladin ou restriction de classe), la boucle ne trouvait
   aucun sort et retombait en boucle sur -1, forçant le joueur à Shift-clic. **Corrigé** : passe 2 de
   repli sur n'importe quel sort castable (`PallyPower_CanBuff`), garantissant qu'un simple clic cycle
   toujours sans se bloquer.
9. **`PallyPower_HasEnoughMana` & vérification Symbol of Kings avant cast** : Branchement complet de
   la table de mana et vérification de `PP_Symbols < 1` en tête de `PallyPowerBuffButton_OnClick` et
   `PallyPower_AutoBless`. Son d'échec `igQuestFailed` et retour visuel explicite au lieu d'un échec
   silencieux ou d'un message trompeur "Couldn't find target". **Corrigé.**
10. **Détection paladin quittant le groupe** : `PallyPower_PaladinLeftGroup` n'itérait que
    `PallyPower_Assignments`. Si un paladin sans assignation partait, le départ était ignoré.
    **Corrigé** : scan de `AllPallys` en plus de `PallyPower_Assignments`.
11. **Smart Rebuff & Mode Compact** : Ajout du rebuff individuel automatique au clic gauche si
    un seul membre manque de buff (économise Symboles des Rois et mana) et option `/pp compact`
    masquant les classes 100% buffées.

### 🟡 Mineurs / qualité — restants (non corrigés, faible priorité)
12. `PP_NextScan = 0` (`MinimapButton.lua:132`) : assigné, jamais lu ailleurs. Reliquat d'un ancien
    mécanisme de throttle remplacé par `uiDirty`/`uiDebounce`. Sans danger, juste mort.
13. `print` redéfini globalement (`PallyPower.lua:108`) avec un commentaire expliquant que c'est pour
    compenser un bug de l'addon "Shagu Tweaks" avec les booléens. Intentionnel, mais c'est un override
    du client entier (affecte tous les autres addons qui appellent `print()`), pas juste PallyPowerTW.
14. `PallyPower_SwapSet` (`MinimapButton.lua:117-136`) : les écritures Aura/Seal/CurrentSet sont à
    l'intérieur de la boucle `for id = 0, 9` alors qu'elles ne dépendent pas de `id` → 10 écritures
    redondantes de la même valeur. Sans impact fonctionnel, juste inefficace.
15. Variables globales involontaires dispersées dans le fichier (`txt`, `cur`, `shift` — ce dernier
    corrigé au §2.7) faute de `local`. Style hérité de l'addon d'origine, cohérent dans son incohérence
    (présent dans beaucoup de fonctions). Pas de risque de crash identifié pour les occurrences
    restantes (`txt`/`cur` sont toujours réassignés avant lecture dans la même passe), donc laissé tel
    quel — un nettoyage global serait un gros diff pour un gain nul en stabilité.
16. `.toc` référençait `Locale\localization-esEs.lua` (casse différente du fichier réel
    `localization-esES.lua`). Fonctionne sur Windows (NTFS insensible à la casse) mais casserait le
    chargement de la traduction espagnole sur un système sensible à la casse. **Corrigé.**

---

## 3. Robustesse réseau

L'addon reçoit des messages d'**autres versions de PallyPower** et d'un **addon tiers (Tactica)**.
Tout handler doit traiter chaque champ comme **non fiable** :
- Toujours `tonumber()` + garde avant arithmétique ou indexation numérique.
- Toujours vérifier l'existence de `AllPallys[sender]` avant d'y écrire des sous-clés.
- Ne jamais supposer qu'un `string.find`/split a réussi (les captures peuvent être nil).
- **Ne jamais écrire `table[x] = valeur` sans avoir vérifié `x ~= nil`** (erreur Lua "table index is
  nil" — c'est la classe de bug trouvée dans le parseur Tactica, §2.5).

**Principe :** un message malformé doit être **ignoré silencieusement**, jamais planter.

---

## 4. Performance

- `OnUpdate` met `uiDirty=true` tant qu'un timer existe → reconstruction UI toutes les `scanfreq`
  (1 s par défaut) en continu pendant un raid (les GB durent 30 min). Acceptable, optimisable
  (ne rafraîchir que les libellés de temps, pas tout le layout) — non fait, risque modéré, à tester.
- `UNIT_AURA` → `ScanOneUnit` ciblé (bon, pas de scan complet à chaque tick de buff).
- `RAID_ROSTER_UPDATE` reconstruit tout le roster + re-scan (correct, peu fréquent).

---

## 5. Plan d'action restant (rien d'urgent)

| # | Action | Risque | Test en jeu requis |
|---|--------|--------|--------------------|
| 1 | Brancher ou supprimer `PallyPower_HasEnoughMana` | Faible si supprimé, moyen si branché | Oui si branché |
| 2 | Optimiser le refresh UI des timers | Moyen | Oui |
| 3 | Nettoyer `PP_NextScan`, redondance `PallyPower_SwapSet` | Très faible | Non |
| 4 | Décider du sort de bénédictions individuelles (voir §0) | — | Discussion produit |

---

## 6. Corrigé — journal

Toutes ces corrections sont du **durcissement pur** (ajout de gardes / validation) sauf la remise à
plat du cast (§0) : elles ne changent pas le comportement nominal, donc faible risque de régression.

**Session précédente :**
- ✅ COOLDOWNS : garde `AllPallys[sender]` avant indexation, sinon `REQ`.
- ✅ VERSION : `tonumber()` + garde nil, comparaison numérique correcte.
- ✅ SELF/ASELF/SSELF : `local` sur `rank`/`talent`/`tmp`/`numbers`, `talent` stocké en nombre, entrées
  vides/invalides ignorées.
- ✅ Affichage talents (grille des paladins) : garde `tonumber(...) or 0`.
- ✅ Boutons grille : `tonumber(pnum/class)` + early-return si la regex échoue.
- ✅ SYMCOUNT stocké en nombre.
- ✅ `local type, id` mort retiré de `OnEvent`.

**Cette session :**
- ✅ **Reset de la logique de cast** vers la version d'origine ivanovlk (voir §0) — plus fiable que
  les itérations successives sur l'override individuel.
- ✅ **Tactica** : garde `n and n ~= ""` sur les 3 branches TANK/HEAL/CLEAR (évite un crash
  "table index is nil" sur message tronqué).
- ✅ **ScanSpells** : garde `elseif RankInfo[id] then` sur la branche Grande Bénédiction (évite un
  crash si l'ordre du grimoire est inhabituel).
- ✅ **`shift` global instable** dans les cycles de bénédiction corrigé (voir §2.7) — bug qui pouvait
  écraser silencieusement toutes les assignations de classe d'un joueur.
- ✅ **`.toc`** : casse du fichier de localisation espagnole corrigée.

### Recommandé mais NON fait (risque/à tester en jeu, faible priorité)
- Standardiser les regex de noms `(.*)` → `(%S+)` sur ASSIGN/AASSIGN/SASSIGN/MASSIGN (déjà
  crash-safe grâce aux `tonumber` de `ddd46f0`, donc gain faible).
- Optimiser le refresh UI des timers.
- Décider du sort de `print` override, `PallyPower_HasEnoughMana`, `PP_NextScan` (§2.8-10).
