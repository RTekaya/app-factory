# Phase 0 — résultats des spikes

2026-10-12 · Claude Code 2.1.296 · abonnement Pro · plugin jetable dans `spikes/plugin/` (chargé avec `--plugin-dir`).
Ce dossier sera supprimé avant la publication.

| Spike | Question | Résultat | Preuve | Conséquence pour l'implémentation |
|---|---|---|---|---|
| S1 | Un subagent peut-il lancer un subagent ? | ✅ Oui | main → `app-factory:outer` → `app-factory:inner` → `Write` ; le fichier marqueur a été écrit par l'agent imbriqué | **Mécanisme A du `block-runner` disponible** dans Claude Code. ⚠ Les subagents tournent en arrière-plan par défaut : le `block-runner` et l'orchestrateur doivent toujours lancer leurs agents en avant-plan (`run_in_background: false`) et attendre leur rapport |
| S1b | Une session enfant `claude -p` peut-elle orchestrer des subagents ? | ✅ Oui (Claude) | S1 lui-même tournait en `claude -p` lancé depuis Bash ; `-p` attend la fin des subagents avant de rendre son résultat | **Mécanisme B disponible.** Options utiles : `--model`, `--effort`, `--json-schema` (sortie structurée validée), `--output-format json` (coût, `modelUsage`, `permission_denials`), `--max-budget-usd` (si plafond) |
| S2 | `plugin.json` accepte-t-il des chemins d'agents personnalisés (`skills/factory/roles/*.md`) ? | ✅ Oui | Agents listés `app-factory:inner` / `app-factory:outer` ; `claude plugin validate --strict` accepte `model`, `effort`, `memory`, `maxTurns`, `tools` | **Une seule source par rôle**, pas de copies `agents/`. Les agents sont appelés `app-factory:<rôle>` |
| S3 | Une règle `deny` gèle-t-elle un fichier en mode auto/bypass ? | ✅ Oui, mieux que prévu | `deny: ["Edit(docs/01-prd.md)"]` bloque Edit **et** Write, en `bypassPermissions`, `auto` et `acceptEdits`, **depuis un subagent**, et même `sed -i` via Bash (refusé, `permission_denials: Bash`) | Le gel des docs dans Claude Code = **une règle `Edit(...)` par fichier**, posée par `approve` et retirée par `change`. Le contrôle portable dans le gate reste pour les autres hôtes |
| S4 | `/factory` fonctionne-t-il tel quel ? | ✅ Oui | `/factory alpha beta` et `/app-factory:factory alpha` déclenchent tous deux le skill ; `$ARGUMENTS`, `${CLAUDE_PLUGIN_ROOT}` et `${CLAUDE_SKILL_DIR}` sont substitués dans le corps du skill | Le README écrit `/factory`. Le skill peut citer ses références et templates par chemin absolu (`${CLAUDE_SKILL_DIR}/references/…`), y compris dans le prompt d'une session enfant |
| S5 | Le skill marche-t-il dans Codex / Gemini / Cursor / Copilot ? | ⏸ Reporté en phase 3 | Gemini CLI 0.26 installé mais pas connecté ; Codex, Cursor et Copilot CLI absents | Fiche Gemini déjà connue : `gemini -p … -m <model> --approval-mode yolo -o json`. **Il te faudra te connecter / installer les autres CLI en phase 3** |
| S6 | Le `model:` d'un skill change-t-il le modèle ? | ✅ Oui | Session lancée en `sonnet`, `/modeltest` (`model: haiku`) → seul `claude-haiku-5-5` consommé | La phase brainstorm peut imposer le niveau `deep` via le frontmatter… mais un frontmatter est fixe : avec des modèles configurables (§3.10), `/factory` vérifiera le modèle de la session et proposera `/model` si besoin |
| S7 | `/factory --unattended` en headless + question en attente | ⏸ À tester en phase 1 | — | Nécessite le vrai skill |
| S8 | `PushNotification` disponible ? | ✅ Présent, y compris en `claude -p` | Visible dans la liste d'outils de l'event `init` | Réception sur téléphone à confirmer avec toi lors d'un vrai run |
| S9 | Nom `app-factory` libre ? | ✅ Oui | `RTekaya/app-factory` n'existe pas ; aucun plugin `app-factory` dans le marketplace officiel | Nom confirmé |
| S10 | Pro : Opus disponible ? message de quota ? | ✅ Opus OK · ⏸ quota | `--model opus` → `claude-opus-5-5` | Le préréglage « Équilibré » marche sur Pro. Message de quota impossible à provoquer sans danger : la détection de `paused_quota` sera écrite de façon défensive (`is_error` + texte « limit ») et vérifiée au premier vrai cas |

## Points de vigilance découverts

1. **Subagents en arrière-plan par défaut** (S1). Sans consigne explicite, un agent parent peut terminer avant ses enfants. Toutes les références d'orchestration diront de lancer chaque rôle **en avant-plan** et d'attendre son rapport.
2. **Pas de paramètre `effort` sur l'outil Agent**. L'escalade « deep + effort high » se fait :
   - en mécanisme B, avec `--effort high` sur la session enfant ;
   - en mécanisme A, avec un rôle d'escalade dédié (`developer-escalated`, `effort: high`) qui renvoie aux mêmes instructions que `developer`.
3. Le mécanisme B donne une **sortie validée par schéma** (`--json-schema`). C'est un vrai contrat machine pour le retour d'un bloc, plus fiable que le mécanisme A. À garder en tête si A pose problème en conditions réelles.
