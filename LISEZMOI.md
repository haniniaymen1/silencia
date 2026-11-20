# Silencia 1.13.29 (FIX 82)

**Application de bureau Windows pour centres d'appels — suppression de bruit IA en temps réel, transcription, et base de connaissances 100 % locale à réponses vérifiées contre les documents.**

Silencia fonctionne intégralement sur le poste de travail : l'audio ne quitte jamais la machine, les documents importés ne quittent jamais la machine, et les modèles d'IA (suppression de bruit, embeddings, LLM de chat) s'exécutent localement. Aucun service cloud, aucune télémétrie, aucun compte.

- Version actuelle : **1.13.29 (FIX 82)**
- Plateforme : Windows 10/11 (application de bureau Electron)
- Licence : MIT (code de l'application)
- Documents accompagnant ce fichier : `README.md` (anglais), `Silencia-1.13.29-Rapport-detaille-Guide-et-Securite.pdf` (rapport détaillé, guide d'utilisation et rapport de sécurité, en anglais)

---

## 1. Ce que fait Silencia

Silencia se place entre le casque de l'agent et le poste de travail et fournit quatre services permanents :

1. **Suppression de bruit en temps réel** — le signal micro est nettoyé par une chaîne IA locale à deux étages (RNNoise + DTLN) avec trois modes d'intensité, annulation d'écho acoustique (AEC), NOTCH adaptatif, AGC et limiteur qualité diffusion. Cible de loudness : −15,6 LUFS ; plafond du limiteur −2,85 dB (ratio 20:1) ; latence ajoutée 24 ms.
2. **Transcription et services vocaux** — reconnaissance de parole locale (Whisper) avec sous-titres et correcteur, et synthèse vocale via les voix SAPI de Windows.
3. **Routage par câble virtuel** — un câble virtuel VB-Cable relie l'audio nettoyé de Silencia au logiciel de visioconférence ; l'installateur peut le déployer, et depuis le FIX 82 cela se fait **uniquement sur clic explicite**, jamais automatiquement au démarrage.
4. **Base de connaissances et chat documentaire (Chat RAG)** — import de fichiers PDF, DOCX, XLSX/XLS, CSV, TXT et MD ; questions en français ou en anglais ; réponses soit rédigées à partir des passages retrouvés (quand le LLM local est disponible), soit fournies en extraits sourcés mot pour mot (mode repli). Chaque réponse cite sa source : nom du fichier et page (ou « document » pour les formats à page unique, « page N (image) » pour les figures OCRisées).

## 2. Nouveauté de la 1.13.29 (FIX 82)

**Suppression totale des popups au premier lancement.** Depuis les premières versions, le câble virtuel était installé automatiquement au démarrage (deux déclencheurs automatiques dans le renderer), ce qui faisait apparaître une fenêtre UAC Windows sans aucun clic. Le FIX 82 supprime les trois déclencheurs automatiques :

- plus d'installation automatique du câble 1,2 s après la bannière « câble requis » ;
- plus de réparation automatique de la branche « périphérique cassé » (comportement v1.10.7) ;
- plus de second déclencheur 2,5 s après le boot.

L'application se lance **directement** (fenêtre + pipeline audio à 350 ms, comme avant), **sans UAC, sans fenêtre d'installateur, sans popup au démarrage**. L'installation et la réparation du câble restent possibles à tout moment depuis les boutons de la bannière dans la fenêtre — l'UAC n'apparaît alors qu'après un clic. Sont conservés : la bannière informative dans la fenêtre (ce n'est pas une popup), les toasts d'information micro/casque/écoute et les boîtes d'erreur honnêtes (démarrage impossible, erreur inattendue). Les 47 autres fichiers body de l'asar sont bit à bit identiques à la 1.13.28.

## 3. Vue d'ensemble des fonctionnalités

### 3.1 Moteur audio temps réel (vérifié bit à bit de la 1.13.23 à la 1.13.29)

| Étage | Détail |
|---|---|
| Pré-nettoyage | RNNoise v2 compilé en WASM — 48 kHz, int16 |
| Débruitage profond | DTLN, deux étages ONNX — 16 kHz, fenêtre d'analyse 512 / pas 128, latence 24 ms |
| Moteur spectral | STFT 512 avec synthèse WOLA, filtrage de Wiener |
| Contrôle ronflette | NOTCH adaptatif, 250–600 Hz |
| Dynamique | AGC (cibles 0,1117 / 0,1061), EQ, limiteur 20:1 à −2,85 dB |
| Ancres de sortie | Makeup 0,994, −2,70 dBFS, −15,6 LUFS, −2,6 dBTP |
| Modes | `normal` / `fort` / `max` — défaut `max`, suppression active, AEC actif, écoute locale coupée au lancement (verrou anti-Larsen) |

Les fichiers du moteur (`dsp-worklet.js`, `nn-engine.js`, `rnnoise.wasm`, `dsp-host.js` hors asar) sont couverts par une base MD5 d'intégrité et ont été vérifiés bit à bit identiques entre 1.13.23 et 1.13.25 sur 10 fichiers audio générés ; aucun FIX depuis n'y a touché.

### 3.2 Base de connaissances et Chat RAG

| Composant | Comportement |
|---|---|
| Formats | PDF (pdfjs-dist, page par page, max 500 pages), DOCX (mammoth, page unique), XLSX/XLS (SheetJS, une page par feuillet, max 20 feuillets × 5000 lignes), CSV/TXT/MD (UTF-8, repli Windows-1252) |
| Images PDF (FIX 79) | Les images embarquées sont extraites (XObject/inline → PNG), filtrées en taille (≥ 32 px, ≤ 8000 px, ≤ 6 Mpx, ≤ 250 par document), dédoublonnées par SHA1, puis OCRisées localement ; un PDF scanné sans couche texte n'est plus rejeté — l'OCR reconstruit le contenu. Les sources sont citées « page N (image) ». |
| Découpage | ~800 caractères, chevauchement 120, fusion des trop courts ; limites 4000 passages/document et 20000/base |
| Indexation | Index DOUBLE : BM25 pur JS (k1 = 1,4, b = 0,75, antilexique FR compact) + vecteurs BGE-M3 (1024 dims) stockés en matrice Float32 plate |
| Recherche | Fusion hybride 0,6 × vecteur + 0,4 × BM25 normalisé, seuils de pertinence exigeants |
| Gardien (rag-v3.js, FIX 74/75/77/78) | Avant rédaction, les passages doivent passer un gardien (cosinus + BM25 + couverture + règle d'existence des identifiants : un token distinctif de la question, ex. un code de référence, doit exister dans les passages remontés, sinon refus) |
| Validation (validateAnswer v2) | Contrôle post-génération : refus des contradictions et vérifications numériques contre les passages sources |
| Modes de réponse | MODE 1 — réponse rédigée par Qwen2.5-3B quand disponible ; MODE 2 — repli EXTRACTION sourcée mot pour mot (toujours disponible, 100 % fidèle à l'index) |
| Réindexation | Instantanée : les textes OCR sont mis en cache par document (clé = hash d'image), la réindexation ne re-OCR jamais |

### 3.3 Modèles d'IA locaux

| Modèle | Rôle | Provenance | Licence |
|---|---|---|---|
| `qwen2.5:3b` | Rédaction du chat (MODE 1) | Équipe Qwen d'Alibaba, via la bibliothèque Ollama | Qwen Research License (usage de recherche non commercial) |
| `bge-m3` | Embeddings, 1024 dims | BAAI (Beijing Academy of Artificial Intelligence), via Ollama | MIT |
| `dtln1.onnx` + `dtln2.onnx` | Amélioration de la parole à deux étages | DTLN — N. Westhausen & B. Meyer, Université de Hambourg, INTERSPEECH 2020 ; export ONNX | académique/open source |
| `rnnoise.wasm` | Étage de pré-débruitage | Xiph.Org / Mozilla (J.-M. Valin) | BSD 3-Clause |

Ollama tourne strictement sur `127.0.0.1:11434`, est démarré automatiquement par Silencia en priorité **BelowNormal** pour ne jamais affamer le thread audio, et peut être installé silencieusement depuis l'application (build officiel Ollama uniquement).

## 4. Installation

1. Lancez `Silencia-Setup-1.13.29.exe` :
   - double-clic pour l'installation interactive ; ou
   - `Silencia-Setup-1.13.29.exe /S` pour l'installation silencieuse ; ou
   - `Silencia-Setup-1.13.29.exe /S /D="D:\Silencia"` pour un répertoire personnalisé.
   L'installateur ferme toute Silencia active, écrit les fichiers, vérifie le moteur, puis lance Silencia (case cochée sur la page finale ; également en mode `/S`).
2. Vérifiez : barre de titre = « Silencia v1.13.29 », badge permanent bas gauche = « Silencia v1.13.29 », `version.txt` à côté de `Silencia.exe`.
3. **Premier lancement (FIX 82) : rien n'apparaît.** La fenêtre et le pipeline audio sont prêts en ~350 ms. Si le VB-Cable n'est pas encore installé, une bannière informative apparaît dans la fenêtre ; l'installation ne se fait qu'au clic sur le bouton de la bannière, et seule la fenêtre UAC standard est montrée (aucune fenêtre CMD/PowerShell, aucun assistant de pilote).

Ollama et les deux modèles texte (`qwen2.5:3b`, `bge-m3`) ne sont pas embarqués : l'onglet Base de connaissances propose une installation silencieuse guidée, ou le tirage des modèles, sur demande. En leur absence, le chat répond quand même en MODE 2 (extraction sourcée) — la fonctionnalité n'est jamais morte.

## 5. Guide d'utilisation rapide

### Audio
- Sélectionnez le micro du casque (Silencia le sélectionne tout seul quand présent) et choisissez le mode : `normal`, `fort` (suppression forte, défaut pour les espaces ouverts) ou `max` (suppression maximale, défaut).
- « J'écoute l'environnement » (écoute locale) reste **coupé** au lancement par conception (anti-Larsen) : ne l'activez qu'avec un casque, jamais avec des haut-parleurs. Silencia refuse de monitorer vers des haut-parleurs ou vers le câble virtuel.
- Si un souffle ou un écho résiduel subsiste, vérifiez le « Écouter ce périphérique » de Windows (Propriétés du micro — onglet Écouter : décocher) et le sidetone matériel du casque : ils contournent Silencia et aucune API Windows documentée ne permet de les couper.

### Transcription et voix
- La parole est transcrite localement (Whisper) avec sous-titres et correcteur ; les voix SAPI lisent les textes à voix haute.

### Base de connaissances et Chat RAG
1. Ouvrez l'onglet « Base de connaissances ».
2. Cliquez sur **Importer** et choisissez un PDF, DOCX, XLSX, CSV, TXT ou MD. La progression est affichée (extraction → indexation → embeddings). Le statut doit finir à `ok` avec le nombre de passages.
3. Posez vos questions en français ou en anglais dans la zone de chat. Les réponses réussies montrent la source citée (« fichier.pdf, page 2 » ; « page 3 (image) » pour les figures OCRisées ; « document » pour DOCX/CSV/TXT). Quand le gardien refuse (question hors corpus ou preuve insuffisante), la réponse est un refus explicite — c'est le comportement anti-hallucination qui fonctionne, pas une panne.
4. Utilisez **Réindexer** après avoir modifié un document sur disque (par ex. après une correction du parseur DOCX/mammoth) — la réindexation est instantanée grâce au cache OCR. Les doublons sont refusés par hash de contenu (« déjà importé »).
5. Le badge IA affiche le statut Ollama ; des boutons proposent l'installation guidée ou le tirage des modèles.

### Commutateurs d'installation et journaux
- `/S` silencieux, `/D="chemin"` répertoire personnalisé.
- Journaux : `main.log` dans le profil utilisateur (traces d'interface via `diag:log`, journal d'installation du câble via `cable:openlog`).

## 6. Bilan de vérification (points marquants)

- **Banc de référence 1.13.23 (23 questions, exécuté côté utilisateur)** : 14/23 conformes. Il a mis en évidence la classe d'hallucination Q15 (identifiant d'un document réattribué à une question hors corpus), le gardien trop strict au cosinus sur des faits réels, et le seuil 0,451 trop permissif sur les pièges.
- **FIX 77/78 (1.13.24/1.13.25)** : gardien v2 (la règle d'existence des identifiants vise directement la classe Q15), garantie de contexte hybride (renfort lexical quand BM25/couverture sont forts — la classe de refus Q1–Q3/Q18/Q19), validateAnswer v2 (refus des contradictions, vérifications numériques), et correction de l'indexation vectorielle par lots dans `rag.js`.
- **Banc bilingue 36 questions (ligne 1.13.23/1.13.25)** : 36/36, avec le backend Ollama exercé réellement (aucune simulation). Le piège `CC-7749` (absent de tous les passages) est refusé.
- **Contrôle de non-régression audio** : 10 fichiers audio générés + `dsp-host.js` MD5 identiques entre 1.13.23 et 1.13.25 ; le moteur audio n'a pas bougé depuis.

## 7. Sécurité en bref (détails dans le PDF)

- **Isolation de contexte** : le renderer ne parle au processus principal qu'à travers le `contextBridge` (`preload.js`) ; pas d'accès Node dans le renderer.
- **Surface IPC** : ~45 canaux en liste blanche (`app:*`, `settings:*`, `cable:*`, `sapi:*`, `asr:*`, `nn:*`, `nn2:*`, `ai:*`, `rag:*`, `chat:*`), tous internes à l'application.
- **Réseau** : le seul point de terminaison est l'API Ollama locale sur `127.0.0.1:11434` ; l'installation silencieuse optionnelle d'Ollama télécharge le build officiel ; l'ouverture d'URL externes est contrôlée par liste blanche (`ai:openUrl`). Aucune télémétrie, aucun analytics, aucun chargement de contenu distant, aucun `eval`.
- **Localité des données** : l'analyse des documents est 100 % locale (« aucune donnée envoyée nulle part » — docparse.js) ; la base de connaissances vit dans `%APPDATA%\Silencia\rag`.
- **Intégrité** : une base MD5 de 1 917 fichiers livrés verrouille le build ; les trois fichiers de modèles audio embarqués sont épinglés par MD5 ; les blobs Ollama sont nommés par sha256 dans le magasin.
- **VirusTotal** : voir la partie E du PDF — méthode, contraintes (limite web de 650 Mo, faux positifs Electron fréquents, avertissement sur l'échantillon public) et statut honnête des verdicts par fichier.

## 8. Historique des versions (ligne 1.13.x)

| FIX | Version | Sujet |
|---|---|---|
| 72 | 1.13.20 | Base de connaissances & Chat RAG restaurés du système v1.10.64 (rag.js + docparse.js + ollama.js, IPC `rag:*`/`chat:*`/`ai:*`), en remplacement de la pile kb/* 1.13.x retirée |
| 74/75 | — | Gardien V3 (`rag-v3.js`) : porte cosinus/BM25/couverture |
| 76 | 1.13.23 | Chat bilingue validé (36/36), audit Ollama, aucune simulation |
| 77 | 1.13.24 | Gardien v2 — durcissement du ChatRAG |
| 78 | 1.13.25 | Garantie de contexte hybride + validateAnswer v2 + correction de l'index vectoriel par lots |
| 79 | 1.13.26 | Exploitation des images des PDF : extraction d'images + OCR local + citations « page N (image) » ; PDF scannés acceptés |
| 80 | 1.13.27 | Correction de l'installateur |
| 81 | 1.13.28 | Correction d'affichage (décalage de l'affichage des fichiers importés) |
| 82 | 1.13.29 | Suppression totale des popups au premier lancement (ni UAC ni installateur au démarrage ; installation du câble sur clic explicite uniquement) |

## 9. Organisation des fichiers (application)

```
Silencia/
├── Silencia.exe, version.txt          # coquille Electron + marqueur de version
├── resources/app.asar                 # corps applicatif (voir ci-dessous)
├── resources/app.asar.unpacked/       # fichiers conservés hors asar
└── renderer/dsp/                      # moteur audio (bit à bit, épinglé MD5)
    ├── dsp-worklet.js, nn-engine.js, dsp-core.js
    ├── dsp-host.js (unpacked, hors asar)
    ├── models/dtln1.onnx, models/dtln2.onnx
    └── rnnoise.wasm
```

Dans l'asar : `main.js` (processus principal Electron, IPC, client Ollama, aides installateur), `preload.js` (contextBridge), `renderer/` (interface, `app.js`, `rag-ui.js`, câblage du worklet audio), `rag.js` (RagStore : découpage, BM25, vecteurs, import/réindexation), `rag-v3.js` (gardien), `docparse.js` (PDF/DOCX/XLSX/CSV/TXT/MD + extraction des images PDF), `ocr.js` (OCR local), `ollama.js` (client localhost), `kb-config.json`, `styles.css`.

## 10. Limites connues (dites honnêtement)

- **DOCX** : mammoth extrait le texte brut uniquement — une seule page « document » ; les tableaux deviennent du texte plat et les images embarquées ne sont pas extraites (la chaîne images/OCR du FIX 79 ne s'applique qu'aux PDF). Une correction de mammoth ne touche que le chemin import/réindexation/Chat-RAG des DOCX.
- **Longueur des extraits UI** : les extraits de sources sont tronqués pour l'affichage ; sur les documents à passage unique (DOCX), les faits en fin de passage peuvent ne pas être visibles dans « Sources utilisées » tout en étant dans l'index et dans la réponse.
- **Licence Qwen2.5-3B** : licence de recherche — pour des déploiements commerciaux en centre d'appels, examinez les termes de la licence Qwen ou remplacez le modèle par un modèle à licence commerciale via Ollama.
- **VirusTotal** : les modèles Ollama multi-Go et le gros installateur dépassent la zone de confort de l'upload web VirusTotal (650 Mo) ; l'intégrité est garantie à la place par l'épinglage MD5/sha256 et la provenance officielle. Voir le PDF, partie E.
