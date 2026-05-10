# 🛡️ Génération Automatique de Règles Snort par Architectures RAG

> **Devoir 3 — NLP 2026 | Sujet 2 : Génération de règles SNORT**

---

## 📌 Description du projet

Ce projet explore la génération automatique de **règles Snort** à partir de descriptions textuelles d'attaques réseau, en comparant **7 architectures RAG** (Retrieval-Augmented Generation) allant de la baseline sans contexte jusqu'à un agent avec boucle de raisonnement itératif.

L'objectif est de mesurer dans quelle mesure enrichir un LLM local avec un contexte de retrieval améliore la qualité syntaxique et sémantique des règles Snort générées.

---

## 📂 Structure du projet

```
Devoir3/
├── Notebooks/
│   ├── 01_dataset.ipynb          # Exploration et visualisation du dataset
│   ├── 02_embeddings.ipynb       # Génération des embeddings + index FAISS
│   ├── 03_baseline_no_rag.ipynb  # LLM sans RAG (référence)
│   ├── 04_rag_classic.ipynb      # RAG classique FAISS
│   ├── 05_rag_rerank.ipynb       # RAG + Cross-encoder (re-ranking)
│   ├── 06_rag_hybrid.ipynb       # Dense + BM25 + RRF
│   ├── 07_rag_multihop.ipynb     # 2 hops + reformulation LLM
│   ├── 08_rag_graph.ipynb        # Graphe de connaissances (NetworkX)
│   ├── 09_rag_agentic.ipynb      # Agent avec boucle de raisonnement
│   └── 10_evaluation.ipynb       # Métriques + tableau comparatif + Gradio
├── Datasets/
│   ├── snort_dataset.csv           # Dataset brut scrapé
│   ├── snort_dataset_clean.csv     # Dataset nettoyé (CSV)
│   └── snort_dataset_clean.json    # Dataset nettoyé (JSON)
├── Embeddings/
│   ├── snort_faiss.index         # Index FAISS (50 vecteurs)
│   ├── snort_embeddings.npy      # Matrice numpy (50 × 768)
│   └── snort_metadata.pkl        # Métadonnées du dataset
├── Results/
│   ├── results_baseline.json
│   ├── results_rag_classic.json
│   ├── results_rag_reranking.json
│   ├── results_rag_hybrid.json
│   ├── results_multi_hop.json
│   ├── results_graph_rag.json
│   └── results_agentic_rag.json
└── Charts/
    ├── 01_distribution.png
    ├── 01_heatmap_Pro_Sev.png
    ├── 02_similarity_matrix.png
    ├── 02_tsne.png
    ├── 04_baseline_vs_rag.png
    └──  04_retrieval_scores.png
    
```

---

## 🗄️ Dataset

Le dataset contient **50 règles Snort**, couvrant une large variété d'attaques réseau.

| Propriété | Valeur |
|---|---|
| Nombre d'entrées | 50 |
| Types d'attaques | 25 (Brute Force, Port Scan, Injection SQL, Ransomware, DDoS…) |
| Protocoles | TCP (78%), UDP (12%), ICMP (6%), ARP (4%) |
| Niveaux de sévérité | `low`, `medium`, `high`, `critical` |
| Longueur moyenne des descriptions | 7 mots |
| Longueur moyenne des règles | 136 caractères |

<img width="1200" height="600" alt="image" src="https://github.com/user-attachments/assets/fa8075a2-48e8-4b0d-a039-2315ae6e1f65" />


Le protocole TCP domine largement le dataset (39 entrées sur 50), avec une concentration forte sur les sévérités `critical` (12 entrées TCP) et `medium` (14 entrées TCP).

---

## ⚙️ Modèles utilisés

| Rôle | Modèle |
|---|---|
| LLM principal | `Qwen/Qwen2-1.5B-Instruct` (local, GPU) |
| Embedding | `sentence-transformers/all-mpnet-base-v2` |
| Re-ranking | `cross-encoder/ms-marco-MiniLM-L-6-v2` |
| Index vectoriel | FAISS `IndexFlatIP` (similarité cosine) |
| Recherche lexicale | BM25Okapi (`rank-bm25`) |
| Graphe | NetworkX (83 nœuds, 196 arêtes) |

---

## 🔬 Architectures comparées

### 1. Baseline (No RAG) — `03_baseline_no_rag.ipynb`
LLM interrogé directement sans aucun contexte. Score Snort moyen : **0.4/5**. Le modèle génère des règles YARA ou des syntaxes invalides.

### 2. RAG Classique — `04_rag_classic.ipynb`
Retrieval FAISS top-3, contexte injecté dans le prompt. Score Snort : **2.0/5**. Toutes les règles commencent par `alert`, mais `sid` est systématiquement absent.

### 3. RAG + Re-ranking — `05_rag_rerank.ipynb`
Bi-encodeur pour le retrieval, cross-encodeur pour réordonner. Le doc pertinent est systématiquement classé premier (score cross > +1.2 vs négatif pour les autres). Score Snort : **2.4/5**.

### 4. RAG Hybride — `06_rag_hybrid.ipynb`
Fusion Dense + BM25 via Reciprocal Rank Fusion (RRF, k=60). Score Snort : **2.0/5**. Le recall est amélioré mais le LLM (`flan-t5-base`) est moins adapté que Qwen2.

### 5. Multi-hop RAG — `07_rag_multihop.ipynb` ⭐
Hop 1 sur requête originale → reformulation LLM → Hop 2 sur requête enrichie → 6 docs de contexte total. Score Snort : **3.4/5**. `sid` présent dans 3/5 règles. **Meilleure architecture.**

### 6. Graph RAG — `08_rag_graph.ipynb`
Graphe de connaissances (50 docs + 33 entités, 196 arêtes), exploration BFS 2 hops. Score Snort : **2.2/5**. Première apparition du champ `content` hex (Q4 : `|FF 53 4D 42 25|`).

### 7. Agentic RAG — `09_rag_agentic.ipynb`
Boucle itérative (max 3 itérations), k variable (3→4→5), auto-évaluation sur 5 critères Snort. Score Snort : **2.2/5**. La boucle converge uniquement pour Q4 (itération 2). Architecture la plus lente (~100s/requête).

---

## 📊 Résultats comparatifs

| Architecture | BLEU | ROUGE-1 | ROUGE-2 | ROUGE-L | Snort Score | Temps moy. |
|---|---|---|---|---|---|---|
| 🥇 **Multi-hop RAG** | **0.2001** | **0.4618** | **0.2618** | **0.4387** | **3.4 / 5** | 62.8s |
| RAG + Re-ranking | 0.1155 | 0.3822 | 0.1682 | 0.3268 | 2.4 / 5 | 31.9s |
| Graph RAG | 0.0653 | 0.3737 | 0.2161 | 0.3209 | 2.2 / 5 | 34.7s |
| Agentic RAG | 0.0702 | 0.3724 | 0.1756 | 0.3152 | 2.2 / 5 | 100.4s |
| RAG Classique | 0.0292 | 0.3028 | 0.0998 | 0.2951 | 2.0 / 5 | 28.7s |
| RAG Hybride | 0.0693 | 0.2535 | 0.0883 | 0.2328 | 2.0 / 5 | 29.5s |
| Baseline (No RAG) | 0.0086 | 0.1148 | 0.0425 | 0.1148 | 0.4 / 5 | 4.1s |

<img width="2674" height="1517" alt="image" src="https://github.com/user-attachments/assets/6bb239b8-e872-485a-9404-747a948b27c9" />

> Métriques calculées sur 5 requêtes × 7 architectures = 35 évaluations. Ground truth : règles Snort de référence rédigées manuellement.

### Progression du critère `sid` à travers les architectures

| Critère | Baseline | Classic | Re-rank | Hybride | Multi-hop | Graph | Agentic |
|---|---|---|---|---|---|---|---|
| `alert` | 0/5 | 5/5 | 5/5 | 5/5 | 5/5 | 5/5 | 5/5 |
| `msg` | 0/5 | 4/5 | 4/5 | 3/5 | 4/5 | 3/5 | 2/5 |
| `sid` | 0/5 | 0/5 | 0/5 | 0/5 | 3/5 | 3/5 | 1/5 |
| `protocole` | 2/5 | 5/5 | 5/5 | 4/5 | 5/5 | 3/5 | 3/5 |
| `content` | 0/5 | 0/5 | 0/5 | 0/5 | 0/5 | 1/5 | 1/5 |

---

## 🖥️ Interface Gradio

Le notebook `10_evaluation.ipynb` lance une interface web permettant de tester toutes les architectures en temps réel.

<img width="1894" height="937" alt="image" src="https://github.com/user-attachments/assets/e3686440-37ca-4c41-9bf2-9ecba54ebc4e" />
<img width="1874" height="665" alt="image" src="https://github.com/user-attachments/assets/b59b1bc4-7712-4cec-92c6-03296494d310" />


```
http://localhost:7862
```

Fonctionnalités :
- Saisie libre d'une description d'attaque réseau
- Sélection de l'architecture RAG parmi les 7 disponibles
- Affichage des documents récupérés, de la règle générée et du score Snort
- Tableau comparatif des métriques moyennes par architecture
- 7 requêtes exemples prédéfinies (dont Log4Shell, cryptomining)

---

## 🚀 Installation

```bash
# Cloner le dépôt
git clone <repo-url>
cd Devoir3

# Créer l'environnement virtuel
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # Linux/macOS

# Installer les dépendances qui existent dans chaque notebook
pip install faiss-cpu sentence-transformers transformers accelerate torch ...
```

Exécuter les notebooks dans l'ordre : `01` → `02` → `03` → … → `10`.
Le notebook `02` génère les fichiers FAISS nécessaires à tous les notebooks suivants.

---

## 🔑 Points clés

- ✅ Le **Multi-hop RAG** est l'architecture gagnante sur toutes les métriques (BLEU, ROUGE-L, Snort Score)
- ✅ La reformulation de requête en deux passes compense mieux les limites du LLM que la complexité structurelle (Graph) ou itérative (Agentic)
- ✅ Le RAG **multiplie par 8.5 le BLEU** de la baseline (0.009 → 0.200)
- ⚠️ Le champ `content` reste le critère le plus difficile à générer — absent dans 33/35 règles
- ⚠️ L'architecture Agentique est la plus coûteuse (~100s) pour des gains limités
- ⚠️ Toutes architectures confondues, `sid` est le critère le moins bien reproduit par le LLM sans instruction explicite

---

## 📈 Conclusion

Ce projet démontre que l'injection de contexte via RAG améliore significativement la génération de règles Snort par un LLM de taille réduite (1.5B paramètres). Le principal goulot d'étranglement n'est pas le retrieval — dont la qualité est bonne dès le RAG Classique — mais bien la **capacité du LLM à respecter une syntaxe formelle stricte**. Un LLM plus puissant (7B+) ou un fine-tuning sur des règles Snort annotées constitueraient les pistes d'amélioration les plus prometteuses.

---

| | |
|---|---|
| **Encadrante** | Ikram Benabdelouahab |
| **Auteur** | El Attari Taki Eddine |
| **Année** | 2025 – 2026 |
