# Exploiter les documents au-delà de l'OCR

## Objectif

Une page numérisée peut fournir davantage qu'une transcription : des métadonnées, des références juridiques, des destinataires, des tableaux et des éléments visuels. Ce document propose des tâches complémentaires pour valoriser les documents arabes avec un VLM, notamment `google/gemma-3-4b-it`.

Les champs ci-dessous reprennent les catégories visibles dans les captures fournies, sans répéter les sections dupliquées. Il s'agit d'une référence de conception, pas d'un extracteur déjà implémenté ni d'un schéma JSON validé automatiquement.

## Séparer les types de résultats

| Type de tâche | Objectif | Exemple |
| --- | --- | --- |
| Transcription | Reproduire exactement les caractères visibles | `content.full_text` |
| Extraction | Copier une information explicite dans un champ | `source.document_number` |
| Classification | Attribuer une catégorie à partir d'indices visibles | `document_classification.type` |
| Description visuelle | Décrire ce qui est observable sur la page | Position et forme d'un cachet |
| Transformation demandée | Produire une traduction ou une synthèse séparée | `content.subject_translation` |

La transcription doit rester indépendante des interprétations. Une traduction ou une classification ne doit jamais être présentée comme du texte original.

## Règles de fidélité

- Conserver l'arabe original, les noms, les chiffres, la ponctuation et les références exactement tels qu'ils apparaissent.
- Ne pas compléter les passages ambigus et ne pas inventer de mots, personnes, dates ou références.
- Utiliser `null` pour un champ scalaire absent ou indéterminé. Dans la transcription, signaler un passage illisible par `[غير مقروء]`.
- Utiliser `[]` lorsqu'aucun élément n'est observé dans une collection sur les pages analysées. Une collection non analysée doit être signalée séparément, plutôt que supposée vide.
- Pour les booléens, `false` signifie une absence observée ; `null` signifie que l'image ne permet pas de décider.
- Distinguer la pagination imprimée du numéro de page dans le fichier PDF, et la date du document des dates citées.
- Une image ne prouve ni l'authenticité d'une signature ou d'un cachet, ni la validité juridique du document. L'origine numérique d'un cachet peut rester indéterminée.
- Décoder un QR code ou un code-barres avec un outil spécialisé avant de renseigner son contenu ; ne pas reconstruire sa valeur à partir de son apparence.

Les types présentés ci-dessous sont proposés pour une future implémentation. Les chemins avec `[]` désignent des objets dans une liste JSON.

## 1. Classification : `document_classification`

Utilité : organiser un corpus, choisir un schéma d'extraction adapté et faciliter la recherche.

| Champ | Type proposé | Description |
| --- | --- | --- |
| `type` | string ou null | Catégorie principale : `official_letter`, `decree`, `regulation`, `report`, `certificate`, etc. |
| `subtype` | string ou null | Variante : décision ministérielle, note interne, etc. |
| `category` | string ou null | Domaine : juridique, administratif, financier, statistique, etc. |
| `primary_language` | string ou null | Langue principale observée. |
| `secondary_languages` | array | Autres langues présentes. |

## 2. Provenance et références : `source`

Utilité : identifier le document et relier les textes qui se citent entre eux.

| Champ | Type proposé | Description |
| --- | --- | --- |
| `issuing_authority` | string ou null | Nom de l'organisme émetteur, en écriture originale. |
| `department` | string ou null | Division ou service explicitement indiqué. |
| `location` | string ou null | Ville ou région indiquée pour l'autorité ; ne pas la déduire de connaissances externes. |
| `document_number` | string ou null | Référence de ce document, exactement telle qu'imprimée. Conserver les zéros et séparateurs. |
| `related_references` | array | Références des autres documents mentionnés dans le texte. |
| `dates.primary_date.date_text` | string ou null | Date principale dans sa forme originale. |
| `dates.primary_date.calendar_type` | string | `Hijri`, `Gregorian` ou `unknown`, selon les indices explicites. |
| `dates.primary_date.date_type` | string ou null | Rôle : `issue_date`, `effective_date`, `received_date`, etc. |
| `dates.primary_date.indicators` | array | Marqueurs visibles, par exemple `هـ` ou `م`. |
| `dates.primary_date.location_in_document` | string ou null | Emplacement : en-tête, corps, pied de page ou tampon. |

Ne pas convertir automatiquement une date hégirienne en date grégorienne dans `date_text`. Une conversion éventuelle doit être un résultat dérivé distinct, avec sa méthode.

## 3. Propriétés de l'image : `physical_properties`

Utilité : repérer les pages difficiles et adapter le traitement ou la revue manuelle.

| Champ | Type proposé | Description |
| --- | --- | --- |
| `page_number` | string ou null | Pagination imprimée : `7`, `7/254`, etc. |
| `total_pages` | integer ou null | Nombre de pages si indiqué sur le document. |
| `image_type` | string ou null | Capture : `digital`, `scanned`, `photographed`, `mixed`, si identifiable. |
| `quality` | string ou null | Lisibilité : `high`, `medium`, `low`, `illegible`. |
| `color_mode` | string ou null | Aspect : `color`, `grayscale`, `black_white`, `mixed`. |
| `has_watermark` | boolean ou null | Présence visible d'un filigrane. |
| `watermark_description` | string ou null | Description du filigrane observé. |
| `has_security_pattern` | boolean ou null | Présence d'un motif de fond compatible avec un motif de sécurité. |
| `security_pattern_description` | string ou null | Description visuelle, sans certification d'authenticité. |
| `orientation` | string ou null | `portrait` ou `landscape`. |

Analyser l'original pour les caractéristiques originales : notre prétraitement transforme les pages en niveaux de gris et modifie leur résolution et leur contraste. Ne pas attribuer ces transformations au document source.

## 4. Cachets : `official_marks.seals[]`

Utilité : localiser les marques et transcrire leur texte lisible.

| Champ | Type proposé | Description |
| --- | --- | --- |
| `organization` | string ou null | Organisme inscrit sur le cachet, en écriture originale. |
| `position` | string ou null | Emplacement sur la page. |
| `description` | string ou null | Description visuelle du cachet. |
| `is_digital` | boolean ou null | Origine numérique seulement si déterminable ; sinon `null`. |
| `shape` | string ou null | Forme : circulaire, ovale, rectangulaire, etc. |

## 5. Codes : `official_marks.barcodes_qr[]`

| Champ | Type proposé | Description |
| --- | --- | --- |
| `type` | string ou null | `barcode`, `qr_code`, `data_matrix`. |
| `position` | string ou null | Emplacement sur la page. |
| `readable_data` | string ou null | Contenu effectivement décodé et vérifié ; sinon `null`. |

## 6. Signataires : `signatures_authorization.signatories[]`

Utilité : indexer les noms et fonctions explicitement associés à une signature.

| Champ | Type proposé | Description |
| --- | --- | --- |
| `name` | string ou null | Nom complet exactement tel qu'écrit. |
| `name_transliteration` | string ou null | Autre écriture du nom uniquement si elle figure aussi dans le document. |
| `title` | string ou null | Fonction ou titre en écriture originale. |
| `signature_type` | string ou null | `handwritten`, `digital`, `stamp`, `printed_name`, `not_present`, si observable. |
| `position` | string ou null | Emplacement de la signature. |
| `role` | string ou null | Rôle explicite : `primary_signatory`, `co_signatory`, `witness`, `approver`, etc. |

Ne pas identifier une personne à partir de la forme de sa signature seule.

## 7. Circuit d'approbation : `signatures_authorization.approval_chain[]`

| Champ | Type proposé | Description |
| --- | --- | --- |
| `step` | integer ou null | Ordre de l'étape lorsqu'il est documenté. |
| `role` | string ou null | `prepared_by`, `reviewed_by`, `approved_by`, `authorized_by`, etc. |
| `name` | string ou null | Personne citée pour cette étape. |
| `title` | string ou null | Fonction citée pour cette étape. |
| `date` | string ou null | Date de l'étape, si présente, sans conversion implicite. |

Ne pas déduire un ordre administratif de la seule disposition graphique des signatures.

## 8. Destinataires et diffusion : `routing_distribution`

Utilité : retrouver les destinataires et les circuits de transmission décrits dans le document.

| Chemin | Type proposé | Description |
| --- | --- | --- |
| `addressed_to[].type` | string ou null | `person`, `department`, `organization`, `position`, `general`. |
| `addressed_to[].name` | string ou null | Destinataire en écriture originale. |
| `addressed_to[].honorific` | string ou null | Formule visible : `معالي`, `سعادة`, `فضيلة`, etc. |
| `carbon_copy[].type` | string ou null | Catégorie du destinataire en copie. |
| `carbon_copy[].name` | string ou null | Nom du destinataire en copie. |
| `forwarded_to[].type` | string ou null | Catégorie du destinataire d'une transmission. |
| `forwarded_to[].name` | string ou null | Nom du destinataire de la transmission. |
| `forwarded_to[].date` | string ou null | Date de transmission indiquée. |
| `file_reference` | string ou null | Code interne de classement ou suivi, distinct de `document_number`. |
| `classification` | string ou null | Mention de classement ou de confidentialité explicitement présente. |

## 9. Contenu : `content`

Utilité : recherche textuelle et accès au texte intégral, avec transformations séparées.

| Champ | Type proposé | Description |
| --- | --- | --- |
| `subject` | string ou null | Objet ou titre original, lorsqu'il est visible. |
| `subject_translation` | string ou null | Traduction uniquement si explicitement demandée ; contenu dérivé. |
| `keywords` | array | Mots-clés issus des thèmes observés, dans la langue originale. La référence suggère 5 à 10 ; ne pas en inventer pour atteindre un quota. |
| `full_text` | string ou null | Transcription complète dans l'ordre de lecture, avec retours à la ligne préservés. |

## 10. Tableaux : `content.tables[]`

| Champ | Type proposé | Description |
| --- | --- | --- |
| `title` | string ou null | Titre ou légende du tableau, en écriture originale. |
| `headers` | array | En-têtes des colonnes, dans leur ordre observé. |
| `rows` | array d'arrays | Une liste par ligne ; chaque cellule garde sa valeur originale. |
| `notes` | string ou null | Notes de bas de tableau ou annotations visibles. |

Préserver les cellules vides et signaler les cellules illisibles. Ne pas déplacer les chiffres entre colonnes. Pour les cellules fusionnées ou les en-têtes multiples, définir une représentation explicite avant l'implémentation.

## 11. Listes : `content.lists[]`

| Champ | Type proposé | Description |
| --- | --- | --- |
| `type` | string ou null | `numbered`, `bulleted`, `lettered`, `arabic_numbered`, `hierarchical`. |
| `items` | array | Éléments dans l'ordre original ; conserver la hiérarchie et les marqueurs. |

## 12. Graphiques : `content.charts[]`

| Champ | Type proposé | Description |
| --- | --- | --- |
| `type` | string ou null | `bar`, `line`, `pie`, `area`, `scatter`, `table`, `mixed`, `other`. |
| `title` | string ou null | Titre ou légende original. |
| `description` | string ou null | Description limitée à ce que montre le graphique. |

Les captures montrent ces trois champs. L'extraction de séries numériques serait une extension à spécifier ; ne pas inventer de valeurs à partir d'un graphique mal lisible.

## Exemple de demande ciblée

Plutôt que de demander tous les champs en une seule génération, commencer par une tâche courte pour limiter le coût et les sorties tronquées :

```text
Analyse uniquement l'image fournie.
Extrais l'autorité émettrice, le numéro de CE document et sa date principale.
Conserve les valeurs dans leur écriture originale, sans correction ni traduction.
Ne confonds pas le numéro du document avec les références citées dans le corps.
Retourne uniquement un objet JSON avec ces clés :
issuing_authority, document_number, date_text.
Pour une valeur absente ou illisible, utilise null. N'invente aucune valeur.
```

Une réponse JSON valide n'est pas une preuve d'exactitude : chaque valeur doit pouvoir être retrouvée sur la page.

## Organisation proposée pour l'implémentation

1. Produire et conserver une transcription indépendante de la page.
2. Demander séparément les métadonnées, les destinataires, les signatures ou les tableaux nécessaires au besoin métier.
3. Associer chaque valeur extraite à sa page et à une preuve textuelle ou une région de l'image. Ces informations de provenance sont une extension proposée, pas des champs visibles dans les captures.
4. Valider le JSON, les types, les listes de catégories et les relations entre champs avec un schéma versionné, puis vérifier la fidélité au scan.
5. Regrouper les résultats au niveau du document en conservant les sources par page. Un PDF peut contenir plusieurs courriers ou décisions : ne pas fusionner leurs références sans vérifier leurs frontières.
6. Évaluer séparément les ajouts, omissions, références, dates, cellules de tableau et erreurs de classification sur des exemples annotés. Pour le finetuning, séparer entraînement et test par document, pas uniquement par page.

Ces tâches permettent d'exploiter davantage les documents tout en conservant la priorité du projet : aucune information inventée dans les données juridiques.

## Prioriser les tâches avec un budget limité

| Étape | Tâches | Pourquoi commencer ici |
| --- | --- | --- |
| 1 | Texte intégral, numéro du document, date originale, autorité | Informations essentielles et vérifiables directement sur la page. |
| 2 | Destinataires, références citées, articles et listes | Recherche et navigation dans le corpus. |
| 3 | Tableaux, signatures et cachets lisibles | Informations structurées nécessitant parfois des recadrages. |
| 4 | Classification, mots-clés, descriptions et traduction demandée | Enrichissements à évaluer indépendamment de la transcription. |

Cette progression est proposée, pas un engagement de performance. Ne demander que les tâches utiles au besoin métier. Mesurer la latence et les erreurs avant de lancer un traitement complet du corpus ou un finetuning.

## Statut des valeurs et preuves

`null` seul ne distingue pas une absence d'une zone illisible. Une extension utile consiste à conserver un registre `field_results` séparé du JSON métier. Chaque entrée décrit un champ sans modifier sa valeur originale.

| Champ de suivi | Rôle |
| --- | --- |
| `field_path` | Chemin du champ métier concerné. |
| `status` | `observed`, `absent`, `unreadable`, `ambiguous`, `not_evaluated`. |
| `evidence.page_index` | Numéro de page dans le PDF, à partir de 1 ; distinct de la pagination imprimée. |
| `evidence.quote` | Extrait exact visible sur la page, pas une justification inventée. |
| `evidence.region` | Rectangle optionnel `[x_min, y_min, x_max, y_max]` normalisé entre 0 et 1. Origine en haut à gauche de l'image identifiée. |
| `review_status` | `pending`, `accepted`, `rejected`. |

Une zone doit être identifiée comme appartenant au scan original ou à une image transformée ; leurs coordonnées ne sont pas interchangeables après recadrage ou rotation. Les preuves produites par le modèle doivent être vérifiées, elles aussi. Un score de confiance déclaré par le modèle ne remplace pas cette vérification.

### Exemple JSON sans données inventées

Cet exemple représente une extraction non exécutée. `page_index` est une métadonnée fournie par le pipeline ; aucune valeur juridique n'est préremplie.

```json
{
  "schema_version": "0.1-proposal",
  "processing": {
    "task": "source_metadata",
    "model_id": "google/gemma-3-4b-it",
    "model_revision": null,
    "prompt_version": "source-metadata-v1",
    "page_index": 1,
    "generation_complete": null
  },
  "source": {
    "issuing_authority": null,
    "document_number": null,
    "dates": {
      "primary_date": {
        "date_text": null,
        "calendar_type": "unknown"
      }
    }
  },
  "field_results": [
    {
      "field_path": "source.document_number",
      "status": "not_evaluated",
      "evidence": {
        "page_index": 1,
        "quote": null,
        "region": null
      },
      "review_status": "pending"
    }
  ]
}
```

L'enveloppe `processing` et le registre `field_results` sont des extensions proposées. Le pipeline doit renseigner les métadonnées d'exécution lui-même : le modèle ne doit pas inventer sa révision, le numéro de page ou le statut de génération.

## Prompts complémentaires

Les exemples suivants décrivent des tâches futures. Ils ne garantissent ni un JSON valide ni une extraction fidèle.

### Références juridiques

```text
Relève uniquement les références à d'autres documents visibles sur cette page.
Pour chacune, retourne reference_text et evidence_quote dans une liste JSON.
Copie les références et citations exactement, sans normaliser les chiffres.
Exclus le numéro propre du document courant. Si la distinction est ambiguë,
signale ambiguous au lieu de décider. N'ajoute aucune référence externe.
```

### Tableau

```text
Transcris uniquement le tableau visible dans l'image.
Retourne title, headers, rows et notes en JSON.
Conserve l'ordre des colonnes, les valeurs originales et les cellules vides.
Une cellule vide contient "" ; une cellule illisible contient null.
Ne calcule pas les totaux et ne complète aucune valeur manquante.
Si la structure est ambiguë, signale-la séparément sans réorganiser les données.
```

### Question sur le document

```text
Réponds à la question uniquement à partir des pages fournies.
Retourne answer, evidence_quote et page_index.
Si la réponse n'est pas explicitement établie, utilise answer: null.
Ne transforme pas une règle générale en conclusion sur une affaire individuelle.
N'utilise aucune connaissance externe pour compléter le document.
```

La réponse à une question est un résultat dérivé : elle doit être conservée séparément de `content.full_text` et accompagnée de preuves vérifiables.

## Pages denses et qualité de l'image

La largeur actuelle de 600 pixels est un point de départ. Pour les petits caractères, comparer plusieurs résolutions avec les mêmes pages et prompts, en enregistrant le coût mémoire et le temps.

- Conserver un rendu original de qualité suffisante pour vérifier les caractères arabes et les diacritiques.
- Recadrer les zones difficiles avec leurs coordonnées, puis restituer leur ordre de lecture. Un peu de contexte autour de la zone évite de perdre les libellés.
- En cas de génération tronquée, augmenter le budget de tokens ou traiter des régions séparées. Ne pas considérer une sortie coupée comme une transcription complète.
- Fusionner les recadrages en conservant leurs provenances et en contrôlant les doublons dans les zones qui se chevauchent.
- Pour les tampons colorés et les codes, comparer avec l'image couleur originale avant de se fier au prétraitement en niveaux de gris.

Ces adaptations nécessitent une implémentation spécifique ; elles ne sont pas présentes dans la conversion actuelle.

## Validation avant utilisation

| Contrôle | Critère |
| --- | --- |
| Syntaxe | JSON analysable ; pas de clés dupliquées ni de texte ajouté autour. |
| Structure | Champs autorisés, types attendus et statut explicite des valeurs inconnues. |
| Références et dates | Correspondance exacte avec la source, sans conversion ou correction implicite. |
| Tableaux | Nombre et ordre des cellules cohérents avec la structure observée. |
| Preuves | Chaque valeur critique est reliée à une page et vérifiée sur le scan. |
| Complétude | Pas de troncature ignorée ; les zones non traitées sont signalées. |

Si le JSON est invalide, conserver la sortie brute pour diagnostic et marquer le résultat comme invalide. Une réparation automatique ne doit jamais créer ou modifier les valeurs juridiques pour satisfaire le schéma.

### Mesurer les résultats

| Mesure | Ce qu'elle évalue |
| --- | --- |
| CER / WER | Distance d'édition au niveau caractère / mot par rapport à une transcription de référence. |
| Ajouts non justifiés | Nombre de champs ou passages contenant des informations absentes de la source. |
| Exactitude des champs | Correspondance exacte pour les dates, noms et références annotés. |
| Couverture | Part des champs présents correctement retrouvés ; distinguer absence et omission. |
| Exactitude des cellules | Valeurs correctes et placées dans la bonne ligne et colonne. |
| Taux de JSON valide | Qualité du format, mesurée séparément de l'exactitude du contenu. |
| Latence et mémoire | Coût pratique par page et par tâche, avec matériel et configuration enregistrés. |

Définir les règles de normalisation avant l'évaluation. Rapporter les résultats exacts séparément des résultats normalisés : une normalisation des chiffres, espaces ou lettres arabes peut masquer une erreur importante. Conserver un jeu de test par document, comprenant scans difficiles et pages denses, sans fuite depuis l'entraînement.

Les seuils d'acceptation doivent être définis avec le besoin métier. Pour une valeur juridique critique, une preuve manquante, une ambiguïté ou un ajout non justifié doit déclencher une revue humaine. Une absence d'erreurs sur quelques exemples ne garantit pas la fidélité sur tout le corpus.

## Traçabilité et prochaines étapes

Enregistrer pour chaque exécution : empreinte du PDF source, page, transformations de l'image, checkpoint et révision, versions des bibliothèques, prompt versionné, paramètres de génération, sortie brute, résultat validé et état de revue. Ne jamais inclure de token Hugging Face dans ces enregistrements.

La prochaine implémentation peut commencer par une seule tâche de métadonnées, un schéma versionné et quelques exemples annotés. Étendre ensuite aux tableaux et aux références si les erreurs observées et le budget le permettent. Ce document reste une proposition technique ; il n'ajoute ni nouvelles inférences au notebook ni validation automatique.
