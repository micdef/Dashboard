# Dashboard

Centralised supervision of a small fleet of Linux and Windows servers: heartbeats, inventory, readable logs, CVE exposure, AI-based anomaly detection and reports, and remote commands without SSH.

[English](#english) · [Français](#français)

---

## English

### Why

Linux logs are hard to read (`journalctl`, Nginx files), and the Windows Event Viewer is slow. Dashboard gathers everything in one place and answers simple questions: which machine has what, which log belongs to which machine, what is abnormal, and what is exposed.

Dashboard is the final project of the AI developer training at Technofutur TIC (2026-2027).

### Features (V1 target)

- **Fleet status**: heartbeats and host state for every supervised machine.
- **Machine page**: operating system, packages, pending updates, CVE exposure, state, load (processor, memory, disks), and the logs of that machine.
- **Logs**: journald, Nginx files, Windows Event Log, and custom formats described by an XML parser definition.
- **AI**:
  - deterministic rules for known attacks;
  - an anomaly model, fed with log templates and metrics, trained in Python (scikit-learn) and run in .NET through ONNX Runtime;
  - a local LLM that writes reports from computed facts, never from raw logs.
- **Remote commands**: a fixed catalogue of typed commands, checked against a local allowlist on each machine, with a tamper-evident audit trail. No SSH, no inbound port on the machines.
- **Accounts and roles** for the web interface.

### Architecture

| Component | Role |
|---|---|
| Agent (.NET Worker Service) | runs on each machine; opens the connection itself (gRPC, mutual TLS) |
| Collector | receives agent data, computes host state, applies rules, delivers commands |
| API | serves the web interface, authorises users, creates commands |
| Enrolment service | private certificate authority for the agents: enrolment, renewal, revocation |
| Revocation list service | serves the signed revocation lists to the agents, independently of the Enrolment service |
| Data gateway | the only process with database access (stored procedures, TVPs, views) |
| Web site (Blazor Web App) | the user interface |
| AI node | Python training and the local LLM; no database access |
| SQL Server | system of record |

Every decision, with its context, alternatives and consequences, is recorded in the [Architecture Decision Records](doc/analyse/adr/README.md).

### Technology

.NET 10 · C# · ASP.NET Core · Blazor · gRPC · ADO.NET (no ORM) · SQL Server 2025 (Standard edition features; SSDT project) · Python (pandas, scikit-learn) · ONNX Runtime · local LLM (llama.cpp or Ollama).

The database code uses only features available in SQL Server **Standard** edition, so it also runs on the free Standard Developer edition for development and test.

### Status

The project is in its **design phase**: the architecture decisions are being finalised, and no code has been written yet. Planned increments:

1. Heartbeat end to end, with agent enrolment
2. Linux logs and metrics, the machine page, accounts and the audit trail
3. AI: data preparation, training, ONNX inference, LLM report
4. Windows agent and XML log parser
5. Rules and CVE
6. Remote commands

### Repository layout

| Path | Content |
|---|---|
| `src/` | source code (to come) |
| `test/` | tests (to come) |
| `doc/analyse/` | analysis and architecture decisions |
| `doc/adm/` | administration and installation (to come) |
| `doc/user/` | user documentation (to come) |
| `doc/assets/` | images and visual references |

### Licence

Dashboard is published under the [PolyForm Strict License 1.0.0](LICENSE): you may use it for any noncommercial purpose, but you may not distribute it or make changes or new works based on it.

### Author

Michaël Defraene, with Claude (Anthropic) as co-developer.

---

## Français

### Pourquoi

Les logs Linux sont difficiles à lire (`journalctl`, fichiers Nginx), et l'Observateur d'événements de Windows est lent. Dashboard rassemble tout au même endroit et répond à des questions simples : quelle machine a quoi, quel log appartient à quelle machine, ce qui est anormal, et ce qui est exposé.

Dashboard est le projet de fin de la formation Développeur IA de Technofutur TIC (2026-2027).

### Fonctionnalités (cible de la V1)

- **État du parc** : heartbeats et état de chaque machine supervisée.
- **Fiche par machine** : système d'exploitation, paquets, mises à jour en attente, exposition aux CVE, état, charge (processeur, mémoire, disques), et les logs de la machine.
- **Logs** : journald, fichiers Nginx, journal d'événements Windows, et formats propres décrits par une définition de parser XML.
- **IA** :
  - des règles déterministes pour les attaques connues ;
  - un modèle d'anomalies, alimenté par les templates de logs et les métriques, entraîné en Python (scikit-learn) et exécuté en .NET avec ONNX Runtime ;
  - un LLM local qui rédige des rapports à partir de faits calculés, jamais à partir des logs bruts.
- **Commandes à distance** : un catalogue fixe de commandes typées, vérifiées par une liste blanche locale sur chaque machine, avec un audit dont toute altération est détectée. Pas de SSH, aucun port entrant sur les machines.
- **Comptes et rôles** pour l'interface web.

### Architecture

| Composant | Rôle |
|---|---|
| Agent (Worker Service .NET) | tourne sur chaque machine ; ouvre lui-même la connexion (gRPC, TLS mutuel) |
| Collector | reçoit les données des agents, calcule l'état des machines, applique les règles, délivre les commandes |
| API | sert l'interface web, autorise les utilisateurs, crée les commandes |
| Service d'enrôlement | autorité de certification privée des agents : enrôlement, renouvellement, révocation |
| Service des listes de révocation | sert aux agents les listes de révocation signées, indépendamment du service d'enrôlement |
| Passerelle de données (very-back-end) | seul processus qui accède à la base (procédures stockées, TVP, vues) |
| Site web (Blazor Web App) | l'interface utilisateur |
| Nœud IA | entraînement Python et LLM local ; aucun accès à la base |
| SQL Server | stockage de référence |

Chaque décision, avec son contexte, ses alternatives et ses conséquences, est consignée (en anglais) dans les [Architecture Decision Records](doc/analyse/adr/README.md).

### Technologies

.NET 10 · C# · ASP.NET Core · Blazor · gRPC · ADO.NET (sans ORM) · SQL Server 2025 (fonctions de l'édition Standard ; projet SSDT) · Python (pandas, scikit-learn) · ONNX Runtime · LLM local (llama.cpp ou Ollama).

Le code de la base n'utilise que des fonctions disponibles dans l'édition **Standard** de SQL Server : il tourne donc aussi sur l'édition gratuite Standard Developer, pour le développement et le test.

### État

Le projet est en **phase de conception** : les décisions d'architecture sont en cours de finalisation, et aucun code n'est encore écrit. Incréments prévus :

1. Heartbeat de bout en bout, avec l'enrôlement des agents
2. Logs et métriques Linux, fiche par machine, comptes et audit
3. IA : préparation des données, entraînement, inférence ONNX, rapport du LLM
4. Agent Windows et parser XML
5. Règles et CVE
6. Commandes à distance

### Organisation du dépôt

| Chemin | Contenu |
|---|---|
| `src/` | code source (à venir) |
| `test/` | tests (à venir) |
| `doc/analyse/` | analyse et décisions d'architecture |
| `doc/adm/` | administration et installation (à venir) |
| `doc/user/` | documentation utilisateur (à venir) |
| `doc/assets/` | images et références visuelles |

### Licence

Dashboard est publié sous la [PolyForm Strict License 1.0.0](LICENSE) : vous pouvez l'utiliser à toute fin non commerciale, mais vous ne pouvez ni le distribuer, ni le modifier, ni créer d'œuvre dérivée.

### Auteur

Michaël Defraene, avec Claude (Anthropic) comme co-développeur.
