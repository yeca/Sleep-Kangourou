# Sleep Kangourou : CLAUDE.md

App menu bar macOS (Swift/SwiftUI) qui détecte l'absence devant l'écran via la webcam. Après X secondes : baisse de la luminosité de l'écran et extinction du rétroéclairage clavier. Au retour : restauration exacte.

## Projet

- Cible : macOS 13+, MacBook. Swift Package Manager, app `LSUIElement` (pas d'icône Dock).
- Distribution directe uniquement : les API de luminosité et de rétroéclairage sont privées (`DisplayServices`, `CoreBrightness`).
- Vie privée : aucune image caméra écrite sur disque ni envoyée. Détection locale par Vision, ~1 image/s.
- Langue : issues, commentaires et UI en français ; identifiants de code en anglais.

## Structure

```
Package.swift                      # partagé (orchestrateur)
Info.plist
Sources/
  App/                             # point d'entrée, MenuBarExtra
  Core/Contracts.swift             # contrats figés (orchestrateur)
  Core/PresenceController.swift    # machine à états
  Presence/                        # caméra + Vision
  Brightness/                      # luminosité écran
  Keyboard/                        # rétroéclairage clavier
  UI/                              # menu bar, réglages
Tests/
Resources/
  AppIcon/AppIcon-1024.png
  Mascot/kangourou.png
```

## Règles communes (toute session Claude Code)

- Une issue = une branche = une PR. Branche : `feat/sleep-kangourou/<tranche-slug>`.
- Ne modifier que les fichiers listés dans l'issue. Besoin hors périmètre : le signaler, ne pas y toucher.
- Ne jamais modifier `Sources/Core/Contracts.swift` ni `Package.swift` hors de la tranche socle.
- Les API privées sont chargées dynamiquement (`dlopen`/`dlsym`) et dégradent proprement : retourner `nil`, ne jamais crasher.
- Tout code qui parle au matériel passe par un protocole de `Contracts.swift` pour rester testable avec des mocks.
- `swift build` et `swift test` doivent passer avant d'ouvrir la PR.
- Commits en français, au format `type: description` (`feat`, `fix`, `chore`, `refactor`, `docs`).

## Mode ORCHESTRATE

Déclenché par : `ORCHESTRATE epic #<n> sur yeca/Sleep-Kangourou`.

### 1. Préparation
1. Lire l'epic (`gh issue view <n> --repo yeca/Sleep-Kangourou`) : contrats, tranches, fichiers réservés, ordre d'intégration.
2. Lister les sous-issues (`tranche`) et vérifier qu'aucun fichier n'est réservé à deux tranches. Conflit : s'arrêter et le remonter.
3. Créer la branche d'intégration `integration/<epic-slug>` depuis `main`.

### 2. Exécution
1. **Socle d'abord, seul.** Traiter la tranche socle (contrats, `Package.swift`), la merger sur la branche d'intégration. Aucune autre tranche ne démarre avant.
2. **Tranches indépendantes en parallèle** : un subagent par tranche, chacun en worktree isolé (`isolation: worktree`), basé sur la branche d'intégration à jour.
3. Chaque subagent reçoit : le texte de son issue, les contrats figés, la liste exacte de ses fichiers, et la consigne de ne rien toucher d'autre.
4. Chaque subagent livre : sa branche `feat/<epic-slug>/<tranche-slug>` avec build et tests verts. Il n'ouvre pas de PR vers `main`.
5. **Contrôleur en dernier** : la tranche d'intégration démarre quand les tranches qu'elle consomme sont mergées.

### 3. Intégration
1. Merger chaque branche de tranche sur `integration/<epic-slug>` dans l'ordre de l'epic, en lançant `swift build && swift test` après chaque merge.
2. Conflit de merge : il révèle un fichier partagé non prévu. Ne pas résoudre à l'aveugle : corriger la répartition, puis reprendre.
3. Cocher la tranche dans l'epic une fois mergée.

### 4. Checklist de cohérence (avant la PR finale)
- [ ] `Contracts.swift` inchangé depuis le socle
- [ ] Aucune tranche n'a touché un fichier hors de son périmètre (`git diff --stat` par branche)
- [ ] Aucune image caméra écrite ni envoyée (grep sur `write`, `URLSession`, `FileManager` dans `Sources/Presence/`)
- [ ] Les API privées dégradent sans crash (sans rétroéclairage, sans écran intégré)
- [ ] Scénario bout en bout validé : absence > délai → écran dimmé et clavier éteint ; retour → valeurs d'origine restaurées
- [ ] `swift build` et `swift test` verts sur `integration/<epic-slug>`

### 5. Livraison
- Ouvrir une PR `integration/<epic-slug>` → `main`, avec la liste des tranches (`Closes #<n>` pour chacune) et le résultat de la checklist.
- Ne pas merger : la validation finale est humaine.

## Garde-fous

- Tranche bloquée ou ambiguë : commenter l'issue et s'arrêter, ne pas improviser sur les contrats.
- Évolution d'un contrat : proposer le changement dans un commentaire de l'epic, attendre la validation, puis appliquer via l'orchestrateur uniquement.
- Aucune dépendance externe sans validation (le projet n'en a aucune pour l'instant).
