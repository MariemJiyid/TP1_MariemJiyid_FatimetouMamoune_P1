# TP1 — SOAP et WSDL
## Partie B — Contract-first
### 1. Questions sur le WSDL et wsimport

**1. Dans ce WSDL, retrouvez l'opération, son message de fault et l'action SOAP à envoyer.**

- Opération : `emprunterLivre`.
- Message de fault : `LivreIndisponibleFault`.
- Action SOAP : `http://supnum.mr/bib/emprunterLivre`.

**2. Quelle est la signature de emprunterLivre dans EmpruntPortType ?**

```java
public EmpruntType emprunterLivre(
    String idAdherent,
    String isbn
) throws LivreIndisponibleFault;
```

**3. Quel type Java a été généré pour dateRetour ? Comparez avec la partie A.**

Le type Java généré pour `dateRetour` est `XMLGregorianCalendar`, car le type XML `xs:date` est associé à ce type Java.

**4. Quel est le nom de la classe d'exception générée, et d'où vient ce nom dans le WSDL ?**

La classe générée est `LivreIndisponibleFault`. Son nom provient du message de fault `LivreIndisponibleFault` défini dans le WSDL et associé à l'élément `LivreIndisponible`.

---

### 2. Questions sur l'implémentation et SchemaValidation

**1. Le WSDL publié est-il celui que vous avez fourni ? Contient-il le motif de l'ISBN et le type xs:date ?**

Avec `wsdlLocation`, le service utilise le WSDL fourni dans le projet. Il contient le motif de validation de l'ISBN défini par `IsbnType` ainsi que le type `xs:date` pour les dates.

**2. Avez-vous dû modifier 01-emprunter.xml pour qu'il fonctionne ici ? Pourquoi ?**

Non, il n'est pas nécessaire de modifier le fichier XML si sa structure correspond au contrat WSDL et si les valeurs envoyées respectent les règles définies dans le schéma.

**3. Sans @SchemaValidation, l'ISBN 978-2-10 est-il refusé ? Et avec ? Quelle Fault obtenez-vous ?**

Sans `@SchemaValidation`, le serveur ne valide pas systématiquement la requête entrante selon le schéma XML. L'ISBN invalide peut donc être accepté par l'implémentation.

Avec `@SchemaValidation`, le schéma XML est vérifié et l'ISBN `978-2-10` est refusé, car il ne respecte pas le motif défini dans le WSDL. Une erreur SOAP de validation est alors retournée.

**4. Retirez wsdlLocation de l'annotation et redémarrez : qu'est-ce qui change dans le WSDL publié ?**

Sans `wsdlLocation`, le WSDL publié est reconstruit à partir des annotations et des classes Java générées. Certaines contraintes définies uniquement dans le WSDL initial, notamment le motif de l'ISBN, peuvent ne plus apparaître dans le contrat publié.

---

### 3. Conclusion — Choisir et rendre

**1. Citez deux différences que vous avez observées entre le WSDL de la partie A et celui de la partie B.**

- Partie A (Code-first) : le WSDL est généré à partir du code Java et de ses annotations.
- Partie B (Contract-first) : le WSDL est écrit en premier et les classes Java sont générées automatiquement à partir de ce contrat.

**2. Si on renomme le champ dateRetour dans le code, quel projet voit son contrat changer ? Quel risque pour les clients ?**

Dans la partie A (Code-first), le contrat WSDL peut changer automatiquement lorsque le code Java est modifié.

Dans la partie B (Contract-first), le contrat initial reste la référence et ne doit pas être modifié simplement parce que l'implémentation Java change.

Le risque pour les clients est une incompatibilité avec le nouveau contrat, qui peut entraîner des erreurs lors des appels au service.

**3. Quelle approche choisiriez-vous pour le service Paiement, appelé par Scolarité (.NET), Bibliothèque (PHP) et le portail (Node.js) ? Et pour un prototype interne de deux jours ?**

Pour le service Paiement, je choisirais l'approche Contract-first, car plusieurs équipes utilisent des langages différents. Le WSDL permet de définir un contrat commun indépendant du langage de programmation.

Pour un prototype interne de deux jours, je choisirais Code-first, car cette approche est plus rapide à mettre en place.

---

## Conclusion générale

Le Contract-first permet de définir un contrat WSDL avant l'implémentation Java. Il facilite l'interopérabilité entre plusieurs technologies et permet de mieux contrôler les échanges SOAP.

Le Code-first permet de développer rapidement un service à partir du code Java, mais le contrat dépend davantage de l'implémentation.