# Comprendre le projet : OCR arabe avec des modèles vision-langage

Ce guide explique les étapes du projet, leur objectif et les choix techniques. Le notebook de travail est `notebooks/ocr-finetune-vlm.ipynb`. Le README présente la documentation technique en anglais ; ce document sert de parcours pédagogique en français.

## 1. Le problème et la stratégie

Nous voulons extraire le texte et les informations de documents juridiques arabes scannés : titres, références, dates, organismes, tableaux et structure. Les scans peuvent être difficiles à lire. Une extraction utile doit conserver les mots et les chiffres visibles, sans inventer de contenu.

Un OCR classique reconnaît principalement des caractères. Un modèle vision-langage, ou VLM, reçoit une image et une instruction et peut produire une transcription ou une réponse structurée. Cette souplesse permet plusieurs tâches sur une même page, mais le modèle peut aussi générer des informations absentes. Le format JSON ne garantit donc pas la fidélité au document.

La stratégie est de préparer les images, observer un modèle de base, obtenir des réponses candidates d’un modèle plus performant, les vérifier, puis adapter un modèle plus petit pour une utilisation locale. Les priorités sont le budget, l’exécution locale après entraînement et la précision documentaire.

## 2. Ce qui existe aujourd’hui

| Partie | État et limite |
| --- | --- |
| Conversion des quatre PDF locaux | Code disponible ; 34 images ont été générées. |
| Extraction avec Gemma local | Code exécuté dans les expériences ; lenteur, erreurs et sorties tronquées ont été observées. |
| Extraction avec Gemini via OpenRouter | Code disponible et requêtes réalisées ; certains essais se sont arrêtés faute de crédits. |
| Images et annotations importées | Corpus présent : 2 059 images dans 71 dossiers et fichier JSONL associé. |
| Formatage du corpus importé | Vérifié : 3 912 exemples d’entraînement et 80 de validation. Ce sont des exemples candidats. |
| Préparation Vast.ai et LlamaFactory | Code et fichiers de préparation disponibles. Aucun entraînement distant n’a été lancé par l’assistant. |
| Checkpoint externe | Cellules de téléchargement ajoutées ; ce checkpoint n’est pas le résultat d’un entraînement effectué dans ce projet. |

Les chiffres d’exemples correspondent au corpus importé vérifié. Ils ne constituent ni une preuve de qualité des labels ni un résultat d’évaluation du modèle.

## 3. L’environnement local

Le projet utilise Python et VS Code avec un notebook Jupyter. Le noyau recommandé est celui de `.venv` dans le projet.

- `.venv` contient l’interpréteur Python et les bibliothèques du projet.
- `requirements.txt` décrit les dépendances.
- `.env` contient les secrets locaux ; il est ignoré par Git.
- `.env.example` décrit les variables attendues, sans véritables clés.

Une commande `%pip install` installe dans le noyau actif. Si le noyau est local, elle n’installe rien sur Vast.ai. De même, louer une machine distante ne change pas automatiquement le matériel utilisé par le notebook.

`os`, `pathlib`, `glob`, `json`, `random` et `zipfile` appartiennent à Python. PyMuPDF rend les pages PDF ; Pillow manipule les images ; tqdm affiche l’avancement ; Transformers et PyTorch servent à charger et exécuter Gemma ; PEFT fournit les outils pour les adapters LoRA.

## 4. Transformer les PDF en images

PyMuPDF rend une page à la fois à 200 DPI. Cela évite de charger toutes les pages simultanément et ne nécessite pas l’installation externe de Poppler.

`preprocess_image` convertit l’image en niveaux de gris, réduit sa largeur si elle dépasse 600 pixels en conservant les proportions, puis augmente le contraste avec un facteur 1,5. `convert_pdf_to_images` sauvegarde les pages en JPEG avec une qualité de 85.

Les images des PDF locaux sont placées dans `outputs/images/<nom-du-PDF>/page_001.jpg`. Le corpus téléchargé est distinct : `pdf_images/0001/page_001.jpg`, par exemple.

Réduire la taille limite les ressources nécessaires, mais peut faire perdre des détails dans les pages denses. Le contraste et les niveaux de gris peuvent aussi modifier des indices visuels. Ces paramètres doivent être évalués sur les documents, pas considérés comme une amélioration garantie.

## 5. Comprendre les deux sources de données

### Les PDF locaux

Les quatre PDF locaux permettent de tester notre conversion et nos premières extractions. Ils ne sont pas les originaux des 71 documents du corpus téléchargé.

### Le corpus Hugging Face

Source : [bakrianoo/arabic-legal-documents-ocr-1.0](https://huggingface.co/bakrianoo/arabic-legal-documents-ocr-1.0/tree/main/data).

Le fichier `outputs/distillation/ocr-images-sft.jsonl` contient une annotation par ligne. Chaque enregistrement associe un identifiant, un nom de PDF, un chemin d’image, le modèle utilisé et sa réponse. Les chemins Google Drive sont remplacés lors du formatage par les chemins locaux vers `pdf_images/`.

Le fichier vérifié contient 2 060 enregistrements et 2 058 chemins d’image uniques. Tous les chemins référencés correspondent à une image locale. Cependant, 62 réponses échouent à la lecture JSON stricte. Les réponses invalides et les doublons sont ignorés par le formatage.

Nous n’avons pas besoin des PDF originaux pour entraîner sur ces couples image–réponse. Le champ `pdf_name` sert à identifier les pages d’un même document. Les PDF originaux seraient utiles pour vérifier la complétude ou récupérer une meilleure résolution.

Les annotations sont générées par un modèle. Elles ne remplacent pas une vérification humaine. La fiche du dépôt annonce une licence Gemma pour le modèle ; les permissions propres aux documents et aux annotations restent à clarifier avant redistribution.

## 6. Évaluer Gemma avant de l’adapter

Nous utilisons `google/gemma-3-4b-it`, un modèle déjà préentraîné et adapté aux instructions. Le processor prépare l’image, le texte et le format de conversation. PyTorch exécute les calculs ; `model.generate` produit la réponse token par token.

Le prompt demande de respecter le texte visible. `do_sample=False` rend la génération déterministe dans les conditions usuelles, sans garantir l’absence d’erreurs. `max_new_tokens=1024` limite la longueur de la réponse : si la limite est atteinte, le JSON ou la transcription peut être incomplet. Le `TextStreamer` affiche progressivement le texte, mais n’accélère pas les calculs.

Sur Apple Silicon, `mps` désigne le GPU du Mac. Avec `device_map="auto"`, le placement dépend de la mémoire disponible. Si des couches apparaissent sur `disk`, leurs poids sont déchargés et doivent être transférés pour les calculs, ce qui peut fortement ralentir la génération. Les observations du projet ont montré un passage d’un modèle entièrement sur MPS à un placement partiel sur disque, accompagné d’une forte utilisation du swap système.

Cette extraction locale est une expérience de référence : on compare ensuite le modèle adapté au modèle de base sur des pages séparées de l’entraînement.

## 7. OpenRouter : une étape optionnelle

OpenRouter permet d’appeler des modèles hébergés, notamment Gemini, sans charger leurs poids sur notre Mac. Nous l’utilisons pour comparer les extractions et éventuellement produire de nouvelles annotations candidates.

Le notebook suit une configuration compatible avec l’API OpenAI :

```python
# La valeur reste une clé OpenRouter.
os.environ["OPENAI_API_KEY"] = openrouter_token
os.environ["OPENAI_BASE_URL"] = "https://openrouter.ai/api/v1"
```

Le préfixe LiteLLM `openai/` sélectionne le protocole client. L’adresse pointe vers OpenRouter ; la facturation dépend donc d’OpenRouter. Une clé API OpenAI payante ne finance pas ces requêtes. Une erreur HTTP 402 demande de vérifier les crédits ou limites de dépenses OpenRouter.

La boucle d’extraction traite des pages, pas des PDF entiers. Une page produit une réponse. Les réponses réussies restent en mémoire et peuvent être réutilisées lors d’une relance dans le même noyau. Redémarrer le noyau ne recharge pas automatiquement les réponses sauvegardées. Les erreurs de page sont signalées ; les erreurs d’accès, de paiement et de limitation de débit peuvent arrêter le traitement.

Les estimations de coût utilisent les nombres de tokens et les tarifs configurés :

```text
coût estimé = tokens entrée / 1 000 000 × tarif entrée
            + tokens sortie / 1 000 000 × tarif sortie
```

Les valeurs configurées sont 0,50 $ par million de tokens d’entrée et 3,00 $ en sortie, cohérentes avec les facturations observées pour Gemini Flash. Elles doivent être actualisées si le modèle ou ses prix changent. L’estimation n’est pas une facture et peut exclure des requêtes échouées.

Pour le corpus importé, cette génération payante est inutile : les réponses sont déjà dans le JSONL. Il est possible de passer directement au formatage.

## 8. Distillation de connaissances et labels

Gemini joue le rôle du professeur ; Gemma est le modèle à adapter. Nous n’importons pas les poids ou les calculs internes de Gemini. Nous utilisons ses réponses comme exemples supervisés : c’est une forme de distillation fondée sur les réponses.

Une annotation doit être comparée à l’image : mots, chiffres, dates, références, informations ajoutées et informations oubliées. Une sortie JSON valide indique seulement que le texte respecte la syntaxe JSON. Elle ne confirme pas les valeurs extraites.

Si le professeur se trompe et que la réponse est utilisée sans correction, l’entraînement peut apprendre cette erreur à Gemma. La stratégie consiste donc à constituer un corpus fiable et à mesurer les résultats, plutôt qu’à supposer qu’un grand nombre de labels suffit.

## 9. Préparer les conversations LlamaFactory

La section `Format Dataset for Finetuning` transforme chaque page acceptée en deux exemples.

| Exemple | Instruction et réponse attendue |
| --- | --- |
| Tâche 1 | Extraire `content` et `structural_elements` : texte et structure de page. |
| Tâche 2 | Extraire les autres champs : classification, source, marques, signatures, références, etc. |

Chaque exemple ShareGPT contient `conversations` et `images`. Le tour `human` contient `<image>` suivi de l’instruction ; le tour `gpt` contient la réponse attendue. Le mot `gpt` est une étiquette du format, pas une obligation d’utiliser un modèle OpenAI.

Les variables `task_1_sft_recored` et `task_2_sft_recored` rendent ces deux exemples explicites. `pop()` récupère puis retire `content` et `structural_elements` du dictionnaire chargé en mémoire. La première tâche reçoit ces champs ; la seconde reçoit ce qui reste. Le JSONL source n’est pas modifié.

Les documents `0012.pdf`, `0005.pdf` et `0011.pdf` sont réservés à la validation. Les deux tâches et toutes les pages d’un document restent dans le même ensemble. Cela évite une fuite d’information entre pages similaires d’un même document. Les ensembles sont ensuite mélangés avec la graine 101.

Le formatage vérifié produit 3 912 exemples d’entraînement et 80 de validation. Chaque page acceptée produit deux exemples. Les fichiers sauvegardés sont `train-v1.json` et `val-v1.json` dans `outputs/datasets/llamafactory-ocr-finetune-data/`.

Un ensemble de test indépendant reste à définir. Si nous évaluons un checkpoint externe sur des documents déjà utilisés pour son entraînement, le résultat ne mesure pas une généralisation indépendante.

## 10. À quoi sert chaque fichier ?

| Fichier ou dossier | Utilité |
| --- | --- |
| `pdf_images/` | Images téléchargées utilisées par les annotations importées. |
| `outputs/distillation/ocr-images-sft.jsonl` | Source du formatage actuel ; à conserver pour recréer les jeux de données. |
| `outputs/distillation/teacher_annotations_*.jsonl` | Sauvegardes des essais API précédents ; non utilisées automatiquement dans le corpus importé. |
| `train-v1.json` et `val-v1.json` | Conversations destinées à l’entraînement et à la validation. |
| `dataset_info.json` ou entrées préparées | Association des noms de jeux de données à leurs fichiers et colonnes dans LlamaFactory. |
| `outputs/ocr-images-v1.zip` | Archive des images pour transfert vers la machine GPU. |
| `outputs/cloud_gpu/` | Préparation locale des fichiers destinés au serveur distant. |
| `outputs/cloud_gpu/finetune/` | Script d’installation, entrées du registre, YAML et explication des fichiers. |

`outputs/` est ignoré par Git. Les fichiers locaux ne sont pas automatiquement envoyés sur GitHub ou Vast.ai. Copier un fichier de configuration ne copie pas les images auxquelles il fait référence.

## 11. Pourquoi Vast.ai avec un notebook local ?

Nous conservons le projet et le notebook dans VS Code localement, mais prévoyons d’exécuter l’entraînement sur une machine Linux avec GPU NVIDIA louée sur Vast.ai. Le Mac sert à préparer, lire et analyser ; le serveur distant effectue les calculs d’entraînement.

La section `Migrate to Cloud GPU` prépare localement un jeu de fichiers publié sur Hugging Face : téléchargement, extraction puis remplacement des chemins d’images par `/workspace/pdf_images/`. Les variables `cloud_train_ds` et `cloud_val_ds` évitent d’écraser les jeux locaux. Attention : les splits publiés et les splits reconstruits localement sont deux sources possibles ; il faut choisir des fichiers et images cohérents pour une exécution donnée.

On transfère l’archive et les JSON édités au serveur via SSH/SCP ou une interface d’envoi. On extrait les images et vérifie les chemins avant l’entraînement. Une connexion VS Code Remote SSH est optionnelle. Un noyau local ne devient pas distant parce qu’une machine a été louée.

Un tarif horaire faible n’assure pas un coût total faible : il faut tenir compte de la VRAM, de la durée, du disque et du transfert. Il faut sauvegarder les checkpoints puis terminer la location ; du stockage conservé peut rester facturé. Aucune instance n’a été créée par l’assistant dans ce parcours.

## 12. Ce que fait réellement le finetuning

Gemma connaît déjà beaucoup de choses grâce à son préentraînement. Nous voulons adapter son comportement aux documents arabes et aux deux tâches d’extraction du projet.

Pendant un entraînement supervisé, le modèle prédit les tokens d’une réponse. Une fonction de perte mesure l’écart avec la réponse attendue. La rétropropagation calcule les ajustements nécessaires, puis l’optimiseur met à jour les paramètres entraînables.

Avec LoRA, la majorité des poids du modèle de base reste gelée. De petites matrices supplémentaires, appelées adapters, sont entraînées. Cela réduit le nombre de paramètres à modifier et les états de l’optimiseur, mais il faut encore de la mémoire pour le modèle de base, les images, les activations et les séquences.

L’objectif est un modèle local qui suit mieux les tâches et extrait plus fidèlement. Il ne consulte pas le JSONL comme une base de données lors d’une nouvelle inférence. Ce processus n’est pas une recherche documentaire ou un système RAG, et ne garantit pas l’absence d’hallucinations.

## 13. Installer et configurer LlamaFactory

Les cellules locales créent les fichiers de préparation suivants :

- `install_finetune.sh` installe les bibliothèques et le checkout LlamaFactory sur le serveur Linux. Il ne lance pas l’entraînement.
- `dataset_info_entries.json` décrit `ocr_finetune_train` et `ocr_finetune_val`, leurs chemins et le format ShareGPT. Les entrées doivent être ajoutées au registre existant, sans effacer les autres datasets.
- `ocr_finetune.yaml` indique le modèle, les datasets, LoRA, les paramètres d’entraînement, les sorties et le suivi.
- `README.md` explique ces fichiers ; le JSON ne contient pas de commentaires, car JSON ne les accepte pas.

La configuration prépare Gemma 3 4B, LoRA de rang 96, les cibles `all`, une longueur maximale de 12 000 tokens, un batch par GPU de 1 et une accumulation de gradients de 8. Elle utilise un taux d’apprentissage de 0,0001, 20 époques et BF16.

Une époque est un passage sur le jeu d’entraînement. Vingt passages peuvent être longs ou provoquer un surapprentissage. Une séquence de 12 000 tokens peut être coûteuse en mémoire. Ce sont des paramètres de départ conservés du workflow fourni, pas une configuration validée sur un GPU choisi. Un essai court doit précéder une exécution complète.

La reprise depuis `checkpoint-100` est commentée car nous n’avons pas ce checkpoint propre au projet. Une reprise doit utiliser un véritable checkpoint compatible. Hugging Face permet le téléchargement du modèle Gemma avec les droits requis ; WandB reçoit les métriques. Les clés sont saisies sur la machine distante et ne doivent pas être intégrées aux scripts ou au notebook.

La commande `llamafactory-cli train` doit être exécutée dans le terminal distant. Le réglage `DISABLE_VERSION_CHECK=1` désactive un garde-fou de version ; il ne répare pas des dépendances incompatibles. L’installation et l’exécution distante restent à vérifier.

## 14. Télécharger un checkpoint externe

La section dédiée utilise `snapshot_download` pour récupérer seulement `checkpoints/checkpoint-1640/` du dépôt publié. Ce dossier peut contenir les poids LoRA et des fichiers d’état d’entraînement volumineux. Le filtre ne télécharge pas tous les poids du modèle de base.

Ce checkpoint appartient au travail publié par l’auteur du dépôt, pas à un entraînement que nous avons réalisé. Il peut servir à une expérience de comparaison. Il faut le modèle de base compatible pour charger l’adapter. Télécharger ne signifie ni charger, ni exécuter, ni évaluer.

## 15. Mesurer l’amélioration avant de conclure

Comparer le modèle de base et le modèle adapté sur les mêmes documents exclus de l’entraînement, avec des références vérifiées. Observer :

- Erreurs de transcription : taux d’erreur de caractères et de mots, ajouts et omissions.
- Fidélité des champs : dates, noms, numéros et références exacts.
- Complétude : texte manquant et sorties coupées par la limite de tokens.
- Structure : JSON lisible et respect des champs attendus.
- Ressources : temps, mémoire et coût.

Les auto-évaluations du modèle, comme « confiance élevée », ne constituent pas une mesure fiable. Le modèle peut déclarer une grande confiance alors que sa réponse contient des erreurs.

L’entraînement, la création de références validées et l’évaluation indépendante restent nécessaires avant d’affirmer une amélioration. Fusionner LoRA et optimiser le service avec vLLM sont des étapes de déploiement envisagées, pas des résultats acquis.

## 16. Parcours conseillé pour continuer

1. Utiliser le noyau `.venv` et les images importées déjà disponibles.
2. Passer directement au formatage du JSONL importé ; ne pas relancer les requêtes OpenRouter sans besoin de nouveaux labels.
3. Examiner et corriger les annotations, puis recréer les splits et conserver leur provenance.
4. Choisir un jeu de fichiers cohérent pour le serveur et préparer l’archive et les chemins.
5. Choisir une instance GPU adaptée, installer LlamaFactory et vérifier les données sur le serveur.
6. Lancer un essai court avant de décider des paramètres et de la durée complète.
7. Évaluer sur des documents indépendants, sauvegarder les adapters et terminer la location.

Le résultat attendu est une capacité d’extraction locale améliorée, appuyée par des mesures. La qualité des exemples et la séparation des documents sont aussi importantes que le choix du GPU ou du modèle.
