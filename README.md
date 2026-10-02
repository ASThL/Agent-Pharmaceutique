# Agent-Pharmaceutique
Ce projet vise à créer un **agent intelligent d'assistance pharmaceutique**. Il automatise la recherche de notices officielles de médicaments sur la base de données américaine **OpenFDA**, en extrait les informations cliniques essentielles et génère des synthèses claires en français à l'aide de l'intelligence artificielle **Google Gemini**.


## 🏗️ Architecture et Pipeline de Données

Le projet repose sur une architecture en **pipeline séquentiel et modulaire**. Les données circulent d'une étape à l'autre selon le schéma ci-dessous :

[ Question Utilisateur ]
          │
          ▼
┌───────────────────────────┐
│  1. get_drug_label()      │ ──> Interrogation de l'API OpenFDA (JSON brut)
└─────────┬─────────────────┘
          │
          ▼
┌───────────────────────────┐
│  2. extract_drug_info()   │ ──> Filtrage, nettoyage et extraction ciblée
└─────────┬─────────────────┘
          │
          ▼
┌───────────────────────────┐
│  3. summarize_drug_info() │ ──> Synthèse LLM (Gemini) & Traduction
└─────────┬─────────────────┘
          │
          ▼
┌───────────────────────────┐
│  4. ask_agent()           │ ──> Détection d'intention via Function Calling
└───────────────────────────┘

 

## ⚙️ Description des Fonctions

Chaque fonction du Notebook remplit un rôle précis dans la chaîne de traitement :

### 1. `get_drug_label(drug_name)`

* **Rôle :** Récupérer la notice officielle brute au format JSON depuis les serveurs de la FDA.
* **Fonctionnement :** Envoie une requête HTTP `GET` à l'API publique d'OpenFDA en ciblant le champ `openfda.generic_name` avec le nom de la molécule transmis en paramètre.
* **Résultat :** Renvoie un dictionnaire Python contenant l'ensemble de la notice réglementaire américaine, ou `None` si la requête échoue (code HTTP 404).

### 2. `extract_drug_info(raw_data)`

* **Rôle :** Nettoyer le JSON et isoler les champs à valeur ajoutée clinique.
* **Fonctionnement :** Navigue de manière sécurisée dans le dictionnaire via la méthode `.get()` pour éviter les erreurs Python (`KeyError`). Elle cible l'index `[0]` de chaque section conformément à la structure standard des réponses OpenFDA.
* **Données extraites :**
* Nom générique (`name`)
* Effets indésirables (`side_effects`)
* Interactions médicamenteuses (`interactions`)

### 3. `summarize_drug_info(med_info)`

* **Rôle :** Rédiger une synthèse compréhensible et traduite.
* **Fonctionnement :** Construit un prompt structuré contenant uniquement le texte extrait et interroge le modèle `gemini-3.1-flash-lite`. Le modèle a pour consigne stricte de résumer en 4 phrases simples, en français, sans jargon inutile et sans introduire d'informations extérieures (*anti-hallucination*).

### 4. `ask_agent(question)`

* **Rôle :** Détecter l'entité médicamenteuse dans la question de l'utilisateur (*Function Calling*).
* **Fonctionnement :** Fournit la fonction `get_drug_label` comme *outil* (`tool`) au modèle Gemini. L'IA analyse la question posée, identifie le médicament mentionné, extrait son nom et génère un appel de fonction (`function_calls`).

 ## 📌 Spécificités Techniques & Contraintes du Système

Voici les caractéristiques clés de l'environnement et de l'API à prendre en compte :

| Élément | Spécificité Technique | Conséquence / Gestion |
| --- | --- | --- |
| **Structure OpenFDA** | L'API OpenFDA encapsule le texte de chaque section sous forme de liste JSON à un seul bloc. | L'accès au bloc principal via l'index `[0]` dans `extract_drug_info` répond exactement au schéma officiel fourni par l'API. |
| **Langue de l'API** | La base OpenFDA répertorie uniquement les dénominations génériques américaines (en anglais). | Les termes recherchés doivent être en anglais (`metformin`, `ibuprofen`). Une recherche avec des accents ou en français provoque un retour d'erreur `404`. |
| **Sensibilité à la casse** | L'API exige une correspondance exacte sur le champ `generic_name`. | L'utilisation de minuscules est recommandée pour éviter les rejets sur les requêtes. |
| **Chaînage de l'Agent** | L'Agent Gemini identifie la fonction à exécuter mais ne réinjecte pas automatiquement le résultat JSON dans le prompt final. | Le script Python assure l'intermédiaire en transmettant le JSON extrait à la fonction de résumé `summarize_drug_info`. |

## 🛠️ Configuration et Prérequis

Pour exécuter ce Notebook :

1. **Environnement :** Google Colab.
2. **Bibliothèques requises :** `requests`, `google-genai`.
3. **Clé d'API :** Une clé `GEMINI_API_KEY` enregistrée dans les *Secrets* de Google Colab (`userdata`).
