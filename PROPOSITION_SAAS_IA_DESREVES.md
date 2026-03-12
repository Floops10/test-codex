# Desrèves OS — Blueprint SaaS IA (v5.0)

## 0) Positionnement produit (prêt à vendre)

**Desrèves OS** est un SaaS multi-tenant de pilotage d’entreprise centré sur un **Agent IA Dirigeant** qui transforme un flux naturel (texte, vocal, fichiers) en décisions, actions et exécution opérationnelle.

**Promesse commerciale :**
- Unifier stratégie, opérations, finance, CRM, collaboration et exécution IA dans une seule plateforme.
- Réduire la charge mentale du dirigeant avec un cockpit quotidien actionnable.
- Offrir une architecture enterprise-ready (audit, rôles, sécurité, multi-tenant) dès le Day 1.

**Différenciation vs Notion / ClickUp / Monday / Linear :**
- IA native transactionnelle (pas seulement assistant rédactionnel).
- Exécution multi-modules en une commande (budget + tâche + rappel + CRM).
- Gouvernance financière et recouvrement intelligent intégrés.
- Couplage natif Projet ↔ Finance ↔ CRM ↔ Calendrier ↔ Documents.

---

## 1) Architecture technique globale

## 1.1 Vue d’ensemble (C4 - niveau conteneurs)
- **Frontend Web App** (Next.js/React + TypeScript) : dashboard dirigeant, kanban, calendrier, Gantt, CRM, notes, whiteboard, extranet client.
- **API Gateway** (REST + WebSocket) : auth, throttling, routing tenant, observabilité.
- **Core Backend** (NestJS/Fastify) : logique métier (projets, tâches, finances, CRM, rôles).
- **AI Orchestrator Service** : STT, NLU, extraction d’intentions, génération d’actions JSON, validation ambiguïtés.
- **Workflow Engine** (Temporal ou n8n self-host) : automatisations longues et robustes (devis accepté, recouvrement, briefs).
- **Integration Hub** : Notion API, Google Sheets API, Microsoft Graph Excel API, email (SMTP/Sendgrid), paiements (Stripe/GoCardless).
- **PostgreSQL (Supabase compatible)** : persistance multi-tenant.
- **Object Storage** (S3/Supabase Storage) : audios, pièces jointes, exports.
- **Queue/Event Bus** (Redis + BullMQ ou NATS) : traitements async (transcription, imports, sync).
- **Monitoring & Audit** (OpenTelemetry + ELK/Grafana) : traçabilité complète des actions IA.

## 1.2 Principes d’architecture
- **Multi-tenant par `tenant_id`** sur toutes les tables métier.
- **Row Level Security (RLS)** + policies strictes tenant-aware.
- **Pattern Outbox** pour fiabiliser synchronisations externes.
- **Idempotence** sur tous les webhooks et actions IA.
- **Soft delete** + journaux d’audit immuables.
- **Optimistic UI** côté front + rollback contrôlé si conflit serveur.

---

## 2) Modèle de données relationnel (PostgreSQL)

## 2.1 Tables cœur identité & sécurité
- `tenants` : id, name, slug, plan, timezone, created_at.
- `users` : id, email, full_name, locale, is_active.
- `memberships` : id, tenant_id, user_id, role (`admin|collaborator|observer`), invited_by, joined_at.
- `api_keys` : id, tenant_id, name, hashed_key, scopes, last_used_at.
- `sessions` : id, user_id, tenant_id, ip, user_agent, expires_at.

## 2.2 Projets & opérations
- `projects` : id, tenant_id, name, code, status, client_id, start_date, end_date, sold_amount, hourly_rate, risk_score.
- `project_templates` : id, tenant_id, name, version, is_default.
- `project_template_tasks` : id, template_id, title, estimate_hours, milestone_label, dependency_key.
- `milestones` : id, project_id, name, due_date, amount_expected, status.
- `tasks` : id, tenant_id, project_id, title, description, status, priority, assignee_id, estimate_hours, due_date, position.
- `task_dependencies` : id, task_id, depends_on_task_id, relation_type (`FS|SS|FF|SF`).
- `time_entries` : id, task_id, user_id, spent_minutes, note, logged_at.
- `kanban_columns` : id, tenant_id, board_type (`project|crm`), project_id nullable, name, position, month_scope nullable.

## 2.3 Finance
- `finance_accounts` : id, tenant_id, name, type (`cash|bank|card|other`), currency.
- `finance_transactions` : id, tenant_id, project_id nullable, account_id, type (`income|expense`), amount, tax_amount, category, occurred_on, source (`manual|excel|sync|ai`).
- `budgets` : id, tenant_id, project_id, budget_amount, alert_threshold_pct.
- `invoices` : id, tenant_id, project_id, client_id, number, status (`draft|sent|paid|overdue|cancelled`), issue_date, due_date, total_amount, balance_due, payment_link.
- `invoice_reminders` : id, invoice_id, level (`courtois|ferme|tres_ferme`), prepared_subject, prepared_body, approved_by nullable, sent_at nullable.

## 2.4 CRM & stakeholders
- `organizations` : id, tenant_id, name, type (`client|supplier|workshop|freelance|other`), status.
- `contacts` : id, tenant_id, organization_id, first_name, last_name, email, phone, notes.
- `crm_deals` : id, tenant_id, organization_id, title, stage (`a_contacter|rdv_pris|devis_envoye|gagne|perdu`), value_estimate, owner_id.
- `crm_activities` : id, tenant_id, deal_id, contact_id nullable, type, note, due_at.

## 2.5 Collaboration & contenu
- `notes_pages` : id, tenant_id, parent_page_id nullable, title, markdown_content, created_by.
- `whiteboards` : id, tenant_id, title, metadata_json.
- `whiteboard_elements` : id, whiteboard_id, type, payload_json, z_index.
- `files` : id, tenant_id, project_id nullable, owner_id, storage_path, mime_type, size_bytes.

## 2.6 IA, workflow, audit
- `ai_conversations` : id, tenant_id, channel (`text|audio`), started_by, context_json.
- `ai_messages` : id, conversation_id, role (`user|assistant|system`), content, token_usage.
- `ai_action_plans` : id, tenant_id, conversation_id, intent, confidence, action_json, requires_confirmation, executed_at.
- `audio_transcripts` : id, tenant_id, file_id, duration_sec, language, transcript_text, summary_markdown.
- `automation_runs` : id, tenant_id, trigger_type, payload_json, status, started_at, ended_at.
- `audit_logs` : id, tenant_id, actor_type (`user|ai|system`), actor_id nullable, entity_type, entity_id, action, before_json, after_json, created_at.
- `integration_connections` : id, tenant_id, provider (`notion|gsheets|m365`), auth_ref, status, last_sync_at.
- `integration_sync_events` : id, tenant_id, provider, direction (`inbound|outbound`), entity_type, entity_id, external_id, status, error_text.

## 2.7 Relations critiques
- Un `tenant` possède N `memberships`, N `projects`, N `transactions`, N `crm_deals`.
- Un `project` possède N `milestones`, N `tasks`, N `invoices`, N `files`.
- Une `task` peut dépendre de N autres tâches via `task_dependencies`.
- Un `deal` en `gagne` peut créer un `project` (mapping conservé dans `crm_deals.project_id` recommandé).

---

## 3) API — Spécification détaillée

## 3.1 Standards API
- Base: `/v1`
- Auth: JWT + refresh + tenant context header `X-Tenant-Id`
- Réponses: `application/json`
- Idempotence: header `Idempotency-Key` sur endpoints sensibles
- Pagination: cursor-based

## 3.2 Endpoints clés

### Auth & tenant
- `POST /auth/login`
- `POST /auth/refresh`
- `GET /tenants/me`
- `POST /tenants/{id}/invite`

### Projets, tâches, planning
- `GET /projects`
- `POST /projects`
- `POST /projects/from-template`
- `GET /projects/{id}/dashboard`
- `POST /projects/{id}/convert-from-deal`
- `GET /tasks?project_id=&assignee_id=&status=`
- `POST /tasks`
- `PATCH /tasks/{id}`
- `POST /tasks/reorder` (drag & drop kanban idempotent)
- `POST /calendar/assign`
- `GET /calendar/workload?date=YYYY-MM-DD`
- `GET /gantt/{project_id}`

### Finance
- `GET /finance/overview`
- `POST /finance/transactions`
- `POST /finance/import/excel`
- `POST /finance/sync/google-sheets/connect`
- `POST /finance/sync/m365/connect`
- `POST /finance/sync/{provider}/pull`
- `POST /finance/sync/{provider}/push`
- `GET /invoices?status=overdue`
- `POST /invoices/{id}/prepare-reminder`
- `POST /invoices/{id}/approve-and-send`

### CRM
- `GET /crm/deals`
- `POST /crm/deals`
- `PATCH /crm/deals/{id}`
- `POST /crm/deals/{id}/move-stage`
- `POST /crm/deals/{id}/win-and-convert`

### IA central
- `POST /ai/messages`
- `POST /ai/audio/upload`
- `POST /ai/audio/{file_id}/transcribe`
- `POST /ai/action-plans/{id}/confirm`
- `GET /ai/daily-brief`
- `POST /ai/insights/query`

### Extranet client
- `POST /client-portals/projects/{id}/generate-link`
- `GET /client-portals/{token}`
- `POST /client-portals/{token}/milestones/{id}/approve`

## 3.3 Contrats API (extraits)

**`POST /ai/messages` request**
```json
{
  "message": "Ajoute 300€ de frais au projet Web et rappelle-moi de facturer demain",
  "channel": "text",
  "context": {
    "timezone": "Europe/Paris"
  }
}
```

**response**
```json
{
  "plan_id": "apl_123",
  "intent": "multi_action_finance_task",
  "confidence": 0.94,
  "requires_confirmation": false,
  "actions_executed": [
    {"type": "create_expense", "status": "ok", "id": "txn_456"},
    {"type": "create_task", "status": "ok", "id": "tsk_789"}
  ],
  "user_message": "C'est fait : dépense de 300€ ajoutée et rappel de facturation créé pour demain."
}
```

---

## 4) Intégrations Excel / Notion (logiques)

## 4.1 Import Excel massif
1. Upload (`.xlsx/.csv`) → stockage objet.
2. Parsing avec typage colonnes + heuristique (montants positifs = revenus, négatifs = dépenses).
3. Mapping guidé utilisateur si ambigu.
4. Dry-run + preview des lignes affectées.
5. Commit transactionnel en base + audit log + rapport d’import.

## 4.2 Sync Google Sheets / M365 bidirectionnelle
- Chaque table syncable possède un `external_id`.
- Jobs périodiques + webhooks provider si disponibles.
- Détection de conflits par `updated_at` + stratégie `last_writer_wins` configurable.
- Event sourcing dans `integration_sync_events` pour rejouer les échecs.

## 4.3 Sync Notion bidirectionnelle complète
- Mapping bases Notion ↔ entités internes (`projects`, `tasks`, `notes_pages`).
- Sur création/modif/suppression interne : event outbox → push Notion.
- Sur webhook Notion : pull delta → reconciliation.
- Règles anti-boucle: hash de payload + source marker.

---

## 5) System Prompt — Agent IA central (version production)

```text
Tu es "Desrèves AI", l'assistant de direction transactionnel d'une plateforme de pilotage d'entreprise.

OBJECTIF
- Comprendre les demandes en langage naturel (texte ou transcription audio).
- Déterminer l'intention métier et les entités.
- Produire un plan d'actions JSON strict.
- Si ambiguïté, demander validation ciblée avant exécution.
- Sinon, exécuter via les outils backend autorisés et renvoyer une confirmation claire.

CONTEXTE FOURNI À CHAQUE TOUR
- tenant_id, user_id, rôle, fuseau horaire.
- données contextuelles: projets actifs, clients, tâches en retard, factures échues, agenda.
- politiques: permissions RBAC, garde-fous financiers.

RÈGLES DÉCISIONNELLES
1) Prioriser sécurité et exactitude sur vitesse.
2) Ne jamais inventer d'ID ni de données absentes.
3) Si la confiance < 0.80 ou conflit d'entité => requires_confirmation=true.
4) Pour toute action financière > seuil tenant, exiger confirmation explicite.
5) Journaliser chaque action avec raison et source.

INTENTIONS SUPPORTÉES
- create_task, update_task, plan_schedule
- create_expense, create_income, budget_check
- create_invoice_reminder, send_invoice_reminder
- crm_log_note, crm_create_activity, deal_convert_to_project
- query_insight, daily_brief_generate
- audio_meeting_extract_actions

FORMAT DE SORTIE (OBLIGATOIRE JSON)
{
  "intent": "string",
  "confidence": 0.0,
  "requires_confirmation": false,
  "clarification_question": "string|null",
  "entities": {},
  "actions": [
    {
      "type": "string",
      "tool": "string",
      "params": {},
      "idempotency_key": "string"
    }
  ],
  "user_confirmation_summary": "string",
  "post_execution_message": "string"
}

EXEMPLES
- "Ajoute 300€ de frais au projet Web et rappelle-moi de facturer demain"
  => 2 actions: create_expense + create_task.
- "Fin de rdv avec client X... envoie bouquet pour ses fiançailles"
  => crm_log_note + create_task + reminder_date_inference.

STYLE DE RÉPONSE UTILISATEUR
- Concis, professionnel, orienté action.
- Toujours confirmer ce qui a été fait et ce qui reste à valider.
```

## 5.1 JSON Schema (action plan)
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["intent", "confidence", "requires_confirmation", "entities", "actions"],
  "properties": {
    "intent": {"type": "string"},
    "confidence": {"type": "number", "minimum": 0, "maximum": 1},
    "requires_confirmation": {"type": "boolean"},
    "clarification_question": {"type": ["string", "null"]},
    "entities": {"type": "object"},
    "actions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["type", "tool", "params", "idempotency_key"],
        "properties": {
          "type": {"type": "string"},
          "tool": {"type": "string"},
          "params": {"type": "object"},
          "idempotency_key": {"type": "string"}
        }
      }
    }
  }
}
```

---

## 6) Pipeline Audio IA (réunion longue + notes vocales)

1. Upload audio (`/ai/audio/upload`) + métadonnées.
2. Transcription Whisper + diarisation optionnelle.
3. Résumé structuré (`Décisions`, `Risques`, `Actions`).
4. Extraction d’actions atomiques (qui/quoi/quand).
5. Score confiance par action.
6. Exécution automatique si confiance haute, sinon validation.
7. Écriture en BDD (`tasks`, `crm_activities`, `invoice_reminders`, etc.).
8. Journal d’audit + notification utilisateur.

**Garde-fous :**
- Déduplication des actions similaires dans une fenêtre temporelle.
- Blocage des actions destructives sans confirmation.
- Versionnage des transcriptions et résumés.

---

## 7) Workflows internes (exemple devis accepté)

**Trigger**: deal stage `gagne` ou webhook signature devis.

**Actions orchestrées**:
1. Création projet via template lié au type de deal.
2. Génération automatique des jalons avec dates calculées.
3. Création facture d’acompte 30% en brouillon.
4. Création tâches initiales + assignations.
5. Notification équipe + update dashboard dirigeant.

**SLO**: exécution < 30s p95, reprise automatique si incident.

---

## 8) Spécifications Front-end (UI/UX + persistance)

## 8.1 Stack recommandée
- Next.js App Router + TypeScript
- TanStack Query (server state) + Zustand (UI state local)
- DnD Kit (kanban/calendrier)
- TipTap/ProseMirror (notes)
- Yjs + WebSocket (collaboration temps réel whiteboard)

## 8.2 Design system (aligné DA transmise)
- **Couleurs imposées :**
  - `Cloud Dancer #F0EEE9` (55%)
  - `Blue Shadow #485460` (25%)
  - `French Roast #58423F` (10%)
  - `Nimbus Cloud #D5D5D8` (5%)
  - `Cashmere Blue #A3B4C8` (5%)
- **Typographie UI :** Poppins exclusivement (sans-serif géométrique).
- **Iconographie / visuels :** minimalisme premium, forts contrastes soft, whitespace généreux.
- **Micro-interactions :** hover systématique, feedback immédiat, skeleton loaders élégants.
- **Transitions :** fade + slide 120–180ms, courbe `cubic-bezier(0.2, 0.8, 0.2, 1)`.

## 8.3 Règles anti-perte de données
- Auto-save de formulaires (draft local + sync serveur).
- Mutation queue offline-ready (replay à reconnexion).
- Sur drag/drop kanban : mise à jour optimiste + rollback sur conflit.
- Soft delete visuel + undo 10 secondes.

## 8.4 Écrans stratégiques
- **Cockpit dirigeant**: urgences, cashflow, projets à risque, recouvrement.
- **Split Inbox/Calendar**: blocage >8h/jour par ressource (alerte rouge).
- **Kanban projet/CRM**: conversion en projet depuis colonne `Gagné` avec modal.
- **Extranet client**: progression macro, livrables, validation d’étapes.

---

## 9) Sécurité, conformité, exploitation

- Chiffrement au repos et en transit.
- RBAC strict + RLS + journal d’audit exportable.
- Sauvegardes automatiques quotidiennes + PITR.
- RGPD: droit d’accès/suppression, DPA, politique de rétention audio configurable.
- Rate limiting + anti-abus API.

---

## 10) Roadmap produit (go-to-market)

## Phase 1 (0-8 semaines) — MVP commercialisable
- Core: Auth, tenants, projets, tâches, finance de base, CRM pipeline.
- IA: commandes texte multi-actions + daily brief.
- Branding premium aligné DA.

## Phase 2 (8-16 semaines) — Différenciation forte
- Audio long format + extraction automatique d’actions.
- Recouvrement intelligent avec approbation en 1 clic.
- Gantt + dépendances + alertes surcharge calendrier.

## Phase 3 (16-24 semaines) — Scale SaaS
- Sync Notion/Sheets/M365 bi-directionnelle robuste.
- Extranet client complet.
- Facturation SaaS, analytics d’usage, onboarding guidé.

---

## 11) Pitch de vente (prêt à copier)

"Desrèves OS n’est pas un outil de gestion de projet de plus. C’est un système de pilotage d’entreprise augmenté par IA qui exécute réellement: il transforme vos réunions en actions, relie vos projets à votre trésorerie, automatise votre recouvrement, et vous livre chaque matin une vision dirigeant claire en 3 points. Conçu pour les TPE/PME exigeantes, il remplace la fragmentation Notion + tableurs + outils disparates par une seule plateforme élégante, auditable et rentable." 
