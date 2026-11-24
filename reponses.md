# TP1 — SOAP et WSDL : Réponses aux questions

---

## Partie A — Code-first

### Étape 1 — Créer et lancer le service

**1. Rôle des annotations `@WebService` et `@WebFault` :**
* `@WebService` transforme la classe Java `EmpruntService` en service Web SOAP exposable.
* `@WebFault` indique que l'exception `LivreIndisponibleException` représente une erreur métier déclarée dans le contrat WSDL (via un élément `<wsdl:fault>`).

**2. Utilité du constructeur vide sur les classes de données :**
* Les classes de données (`Emprunt`, `LivreIndisponibleInfo`) doivent posséder un constructeur sans paramètre afin de permettre au moteur JAXB (sérialisation/désérialisation XML $\leftrightarrow$ Java) d'instancier dynamiquement les objets par réflexion avant d'injecter leurs attributs.

---

### Étape 2 — Lire le WSDL généré

**1. URL du service et opérations :**
* **URL du service :** `http://localhost:8080/bib/EmpruntService`
* **Opérations :**
  * `emprunterLivre` : message d'entrée `emprunterLivre` et message de sortie `emprunterLivreResponse`.
  * `rendreLivre` : message d'entrée `rendreLivre` et message de sortie `rendreLivreResponse`.

**2. Style, use et version SOAP du binding :**
* **Style :** `document`
* **Use :** `literal`
* **Version SOAP :** `SOAP 1.1` (espace de noms `http://schemas.xmlsoap.org/wsdl/soap/`).

**3. Types XSD générés et validation de l'ISBN :**
* **Type XSD de `isbn` :** `xs:string`
* **Type XSD de `dateRetour` :** `xs:string`
* **Règle de format ISBN :** Non, la regex de validation définie en Java n'apparaît pas dans le schéma XSD généré.

**4. Préfixes d'espace de noms sur les paramètres dans la requête SoapUI :**
* L'espace de noms cible du service est `http://supnum.mr/bib`. Les éléments racines des opérations (`<bib:emprunterLivre>`, `<bib:rendreLivre>`) portent ce préfixe.
* En revanche, les paramètres internes (`idAdherent`, `isbn`, `numeroEmprunt`) sont déclarés localement dans le XSD sans `elementFormDefault="qualified"`. Ils sont donc non qualifiés par défaut et ne portent pas de préfixe d'espace de noms.

**5. Rôle des standards dans la pile Web Services :**
* **HTTP :** Assure le transport des messages sur le réseau entre le client et le serveur (via des requêtes `POST`).
* **XML :** Permet de structurer et représenter les données échangées de manière standardisée.
* **SOAP :** Définit le format d'enveloppe des messages échangés (`Envelope`, `Header`, `Body`, `Fault`).
* **WSDL :** Décrit le contrat technique du service Web (opérations disponibles, types de messages, protocole de binding et adresse d'accès).

---

### Étape 3 — Envoyer des messages SOAP

**1. Pourquoi utiliser `POST` même pour une simple lecture ?**
* Parce que le client doit transmettre une enveloppe XML SOAP complète dans le corps de la requête HTTP pour décrire l'opération et ses paramètres, ce que la méthode `GET` ne permet pas.

**2. Le bloc `<bib:jeton>` du Header a-t-il empêché le traitement ? Pourquoi ?**
* Non, car le service ne déclare aucun gestionnaire (*Handler*) pour cet en-tête et l'attribut `soap:mustUnderstand="1"` n'est pas présent : l'en-tête est donc facultatif et ignoré par le serveur.

**3. Réponse du service au message SOAP 1.2 :**
* Le serveur renvoie une erreur de version (`VersionMismatch` / code HTTP 500), car le endpoint est configuré avec un binding **SOAP 1.1** et rejette les enveloppes utilisant l'espace de noms SOAP 1.2.

**4. Les cinq erreurs du message et leurs corrections :**

| Erreur dans le message initial | Correction apportée | Déclenche une erreur serveur ? |
| :--- | :--- | :--- |
| `soap:envelope` commence par une minuscule. | Remplacer par `soap:Envelope` (respect de la casse XML). | **Oui** (XML invalide / balises non correspondantes). |
| Espace de noms `http://supnum.mr/bibliotheque` incorrect. | Remplacer par `http://supnum.mr/bib`. | **Oui** (opération introuvable dans le service). |
| Balise de fermeture `</rendreLivre>` sans préfixe. | Fermer avec `</bib:rendreLivre>`. | **Oui** (erreur de syntaxe XML). |
| Le `<soap:Header>` est placé après le `<soap:Body>`. | Placer `<soap:Header>` avant `<soap:Body>`. | **Oui** (non-respect du schéma de l'enveloppe SOAP). |
| Paramètre nommé `<bib:numero>` au lieu de `numeroEmprunt`. | Utiliser `<numeroEmprunt>` (sans préfixe et avec le bon nom). | **Non** : JAX-WS ignore les éléments non reconnus et passe `null` à la méthode Java. |

---

### Étape 4 — Provoquer et construire des Faults

**1. Codes reçus pour les cas 1 et 2 :**
* **Cas 1 (ISBN invalide `978-2-10`) :**
  * **Code HTTP :** `500 Internal Server Error`
  * **`faultcode` :** `S:Client`
  * **`faultstring` :** `ISBN mal formé : 978-2-10`
* **Cas 2 (Livre indisponible `978-2-10-074444-4`) :**
  * **Code HTTP :** `500 Internal Server Error`
  * **`faultcode` :** `S:Server`
  * **`faultstring` :** `Livre indisponible`

**2. Contenu de `<detail>` et description dans le WSDL :**
* Dans le cas 2, la balise `<detail>` contient l'élément métier `<ns2:LivreIndisponible xmlns:ns2="http://supnum.mr/bib">` composé de `<isbn>978-2-10-074444-4</isbn>` et `<retourPrevu>2026-12-02</retourPrevu>` (ainsi qu'une trace technique interne `<ns2:exception>` ajoutée par le moteur Metro).
* Sa structure est décrite dans le schéma XSD importé par le WSDL (`?xsd=1`) via le type complexe `LivreIndisponibleInfo` et l'élément `LivreIndisponible` (générés à partir de `@WebFault`), et reliée à l'opération via `<fault name="LivreIndisponibleException">` dans le `<portType>` et le `<binding>`.

**3. Choix du code de Fault par JAX-WS pour le livre indisponible :**
* JAX-WS a choisi le code **`Server`** (`S:Server`), car par défaut toute exception Java personnalisée annotée avec `@WebFault` est mappée comme une erreur côté serveur.
* Il s'agit pourtant d'une **erreur métier liée aux données fournies par le client** (le client tente d'emprunter un ouvrage déjà sorti alors que le serveur fonctionne normalement). Le code **`Client`** (ou **`Sender`** en SOAP 1.2) aurait été plus pertinent.

**4. Comportement de `soap:mustUnderstand` :**
* **Avec `soap:mustUnderstand="1"` :** Le serveur rejette immédiatement la requête avant même d'exécuter la méthode Java et renvoie une Fault SOAP avec `<faultcode>S:MustUnderstand</faultcode>`, car aucun gestionnaire (*Handler*) côté serveur n'est configuré pour traiter l'en-tête obligatoire `<bib:jeton>`.
* **Avec `soap:mustUnderstand="0"` :** L'en-tête est considéré comme facultatif ; le serveur l'ignore et traite l'appel à l'opération normalement.

**5. Différences entre la Fault SOAP 1.2 et les Faults SOAP 1.1 reçues :**
* **Espace de noms :** `http://www.w3.org/2003/05/soap-envelope` en SOAP 1.2 contre `http://schemas.xmlsoap.org/soap/envelope/` en SOAP 1.1.
* **Structure et nommage des balises (hiérarchique et PascalCase en 1.2) :**
  * `<faultcode>Client</faultcode>` devient `<soap:Code><soap:Value>soap:Sender</soap:Value></soap:Code>` (le terme `Client` est remplacé par `Sender`).
  * `<faultstring>` devient `<soap:Reason><soap:Text xml:lang="fr">...</soap:Text></soap:Reason>` avec l'attribut obligatoire `xml:lang`.
  * `<detail>` prend une majuscule et devient `<soap:Detail>`.

---

## Partie B — Contract-first

### Étape 5 — Générer le code depuis le WSDL

**1. Opération, message de fault et action SOAP dans le WSDL :**
* **Opération :** `emprunterLivre`
* **Message de fault :** `LivreIndisponibleFault`
* **Action SOAP :** `http://supnum.mr/bib/emprunterLivre`

**2. Signature de `emprunterLivre` dans `EmpruntPortType` :**
```java
public EmpruntType emprunterLivre(
    String idAdherent,
    String isbn
) throws LivreIndisponibleFault;