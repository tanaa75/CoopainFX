# Étude de cas – Coopain

## Mission 1 – Gestion des demandes d’insémination 

**1.1**  
Sur cette requête SQL, on peut voir que dans la deuxième partie, il sélectionne la date `MIN`, donc la date la plus ancienne. On fait ça pour récupérer en priorité le lot de paillettes le plus vieux pour un taureau et un type de paillette donnés, à la condition stricte qu'il reste encore des paillettes disponibles dans ce lot (comme l'indique la ligne `stockRestant >= 1`).

**1.2**  
La date de collecte provient de l'attribut `date` de la table `Lot`. Elle nécessite une transformation pour être convertie au format spécifique "AAJJJ".
* **N° d'agrément du CCS** : Ne provient pas de la base de données actuelle ; c'est une constante (le numéro de Coopain) inscrite en dur dans le système, sans transformation nécessaire.
* **N° national d'identification du taureau** : Correspond à l'attribut `idNational` de la table `Taureau`. Aucune transformation n'est requise.
* **Code à barres** : Nécessite l'extraction de trois données : l'attribut `numeroIE` de la table `Taureau`, ainsi que les attributs `date` et `numero` de la table `Lot`. Ces éléments subissent une transformation puisqu'ils doivent être concaténés et encodés au format "128 CRT".
* **N° IE du taureau** : Provient directement de l'attribut `numeroIE` de la table `Taureau`, sans aucune transformation.
* **Quantième de la date de collecte** : Utilise l'attribut `date` de la table `Lot`. Il nécessite une transformation mathématique consistant à calculer le nombre de jours écoulés entre le 01/01/2002 et cette date.
* **Numéro de lot de prélèvement** : C'est l'attribut `numero` de la table `Lot`, affiché tel quel sans transformation.
* **Code de la race** : Récupéré via l'attribut `codeRace` de la table `Taureau` (ou code de la table Race), sans transformation.
* **Nom du taureau** : Provient de l'attribut `nom` de la table `Taureau`. La donnée en elle-même ne subit pas de transformation, elle est simplement mise en gras lors de l'impression.

**1.3**  
```sql
SELECT TypePaillette.libelle, Tarif.prixPaillette 
FROM Tarif 
JOIN TypePaillette ON Tarif.idTypePaillette = TypePaillette.id 
WHERE Tarif.idNationalTaureau = 'FR0103015562' ;
```

**1.4**  
```sql
SELECT Taureau.idNational, Taureau.nom, SUM(Lot.stockRestant) AS stock_total
FROM Taureau t
JOIN Lot ON t.idNational = Lot.idNationalTaureau
GROUP BY t.idNational, t.nom
ORDER BY stock_total ASC;
```

**1.5**  
*Tables existantes conservées :*
* `Taureau` (<u>idNational</u>, nom, numeroIE, #codeRace)
* `TypePaillette` (<u>id</u>, libelle)
* `Lot` (<u>#idNationalTaureau</u>, <u>date</u>, <u>numero</u>, nombrePaillettes, stockRestant, #idTypePaillette)

*Nouvelles tables :*
* `Secteur` (<u>idSecteur</u>, nom, description)
* `Vehicule` (<u>immatriculation</u>, marque, modele, dateAchat)
* `Adherent` (<u>idAdherent</u>, adresse, #idSecteur)
* `Vache` (<u>idNationalVache</u>, nom, #idAdherent)
* `Inseminateur` (<u>numeroLicence</u>, #idSecteur, #immatriculationVehicule)
* `Demande` (<u>numeroDemande</u>, dateDemande, dateInsemination, heureInsemination, temperatureAmbiante, compteRendu, #idAdherent, #idNationalVache, #idNationalTaureauChoisi, #idTypePaillette, #numeroLicenceInseminateur, #idNationalTaureauLot, #dateLot, #numeroLot)

*Contraintes d'intégrité (Clés étrangères) :*
* `Adherent` : `#idSecteur` référence `idSecteur` de `Secteur`
* `Vache` : `#idAdherent` référence `idAdherent` de `Adherent`
* `Inseminateur` : `#idSecteur` référence `idSecteur` de `Secteur`, `#immatriculationVehicule` référence `immatriculation` de `Vehicule`
* `Demande` :
  * `#idAdherent` référence `idAdherent` de `Adherent`
  * `#idNationalVache` référence `idNationalVache` de `Vache`
  * `#idNationalTaureauChoisi` référence `idNational` de `Taureau`
  * `#idTypePaillette` référence `id` de `TypePaillette`
  * `#numeroLicenceInseminateur` référence `numeroLicence` de `Inseminateur`
  * `#idNationalTaureauLot`, `#dateLot`, `#numeroLot` référencent la clé primaire composite de `Lot`

---

## Mission 2 – Nouvelles fonctionnalités de l’application Android MobiSemin

**2.1**  
Le kilométrage de fin de tournée saisi est inférieur ou égal au kilométrage de début de tournée. Le système en informe l'utilisateur (message d'erreur) et retourne à l'étape 3 (pour afficher à nouveau le formulaire de saisie des kilomètres afin que l'utilisateur corrige).

**2.2**  
```java
strReq = "CREATE TABLE histoKm(" +
         "id INTEGER PRIMARY KEY AUTOINCREMENT, " +
         "dateTournee TEXT, " +
         "kmDebut real, " +
         "kmFin real, " +
         "idVehicule INTEGER, " +
         "idInseminateur INTEGER)";
db.execSQL(strReq);
```

**2.3**  
En cas de vol de l'ordiphone, la base locale non chiffrée compromet la confidentialité, car un tiers peut lire toutes les données sensibles en clair. L'intégrité est également menacée : le voleur peut falsifier les données locales (comme les kilomètres), ce qui corrompra le système central lors de la synchronisation. Pour protéger l'appareil, il faut chiffrer la base de données avec un outil comme SQLCipher et déployer une solution MDM pour pouvoir effacer les données à distance.

**2.4**  
Manipuler un mot de passe en clair est une faille critique, car n'importe qui accédant à la base de données ou interceptant le réseau peut le lire et usurper l'identité de l'inséminateur. Pour sécuriser l'authentification, il est indispensable de hacher le mot de passe avant son stockage, d'imposer le protocole HTTPS pour chiffrer les échanges sur le réseau, et d'utiliser des jetons de connexion plutôt que de renvoyer le mot de passe à chaque requête.

---

## Mission 3 – Gestion des tournées des inséminateurs

**3.1.1**  
Ce mécanisme s'appelle la surcharge de méthode (ou overloading en anglais). Il permet de définir plusieurs méthodes qui portent exactement le même nom (ici *CATournee*) à l'intérieur d'une même classe, à la condition stricte que leurs signatures soient différentes, c'est-à-dire qu'elles ne doivent pas prendre le même nombre ou le même type de paramètres.

**3.1.2**  
```java
public String getCoordGPS () {
    return this.latitude + " " + this.longitude;
}
```

**3.1.3**  
```java
public ArrayList<Adherent> getAdherents() {
    ArrayList<Adherent> lesAdherents = new ArrayList<Adherent>();
    for (Visite uneVisite : this.laTournee.getLesVisites()) {
        lesAdherents.add(uneVisite.getLeAdherent());
    }
    return lesAdherents;
}
```

**3.1.4**  
```java
public float montantAFacturer() {
    float montantTotal = 0.0f;
    for (PrestationVisite prestation : this.lesPrestationsVisite) {
        int nbActes = prestation.getNombreActes();
        float prix = prestation.getLeTypePrestation().getPrixForfaitaire();
        montantTotal += (nbActes * prix);
    }
    return montantTotal;
}
```

**3.2**  
```java
public class GestionTourneeTest {
    private Inseminateur ins1;
    private Adherent adh1, adh2;
    private Visite v1, v2;
    private TypePrestation tp1, tp2;
    private Tournee t1;
    private GestionTournee gt;

    @Before
    public void setUp() throws Exception {
        tp1 = new TypePrestation("Insémination", 100);
        tp2 = new TypePrestation("Echographie", 10);
        ins1 = new Inseminateur("Petit", "Ferdinand", "0600000100");
        adh1 = new Adherent("Duboeuf", "Georges" , "0600000020", "48.5331", "7.72");
        adh2 = new Adherent("Cow", "Marguerite", "0600000003", "48.5833", "7.75");

        t1 = new Tournee(new SimpleDateFormat("dd-MM-yyyy").parse("06-05-2020"), ins1);
        gt = new GestionTournee(t1);
        v1 = new Visite(adh1, "09:00");
        v1.ajouterPrestationVisite(tp1, 3);
        v1.ajouterPrestationVisite(tp2, 1);
        v2 = new Visite(adh2, "10:30");
        v2.ajouterPrestationVisite(tp1, 1);
    }

    @Test
    public void testCATournee() {
        assertEquals(0.0f, gt.CATournee(), 0.001f);
        gt.ajouterVisite(v1);
        assertEquals(310.0f, gt.CATournee(), 0.001f);
        gt.ajouterVisite(v2);
        assertEquals(410.0f, gt.CATournee(), 0.001f);
    }
}
```

**3.3**  
Une attaque de type Man-in-the-Middle (l'homme du milieu) consiste pour un pirate à s'interposer secrètement entre l'application mobile et le serveur lors de la synchronisation via Internet. L'attaquant peut ainsi intercepter, lire en clair ou même altérer les données sensibles en transit avant de les relayer, compromettant totalement la sécurité du système.
Pour empêcher cette interception et garantir la confidentialité ainsi que l'intégrité des échanges, il est impératif d'exiger l'utilisation du protocole standard HTTPS, qui s'appuie sur TLS pour chiffrer les données de bout en bout.

---

## Comprendre l'architecture et les choix techniques

**1. Traçabilité des paillettes :**  
Il faut identifier le taureau (N° national, race), le lot (date de congélation) et le centre de production pour garantir l'origine et le suivi sanitaire.

**2. Choix de SQLite pour MobiSemin :**  
C'est une base de données légère et embarquée, sans serveur, idéale pour stocker les données localement et permettre à l'application de fonctionner hors-ligne (fréquent en zone rurale).

**3. Risques liés à la perte du Smartphone :**  
* **Confidentialité** : Vol des données personnelles des éleveurs (RGPD).
* **Intégrité** : Modification frauduleuse des données avant synchronisation avec le serveur.

**4. Stockage des mots de passe :**  
Le stockage en clair permet le vol massif d'identifiants en cas de fuite. La solution est le hachage cryptographique (ex: BCrypt) avec un "sel" (salt) aléatoire pour empêcher le déchiffrement.

**5. Attaque "Man-in-the-Middle" :**  
L'attaquant s'intercale entre l'application et le serveur (ex: via un Wi-Fi public) pour intercepter, lire ou modifier les données de synchronisation en transit si elles ne sont pas chiffrées (HTTPS).

**6. Bénéfices d'un ORM comme Hibernate :**  
Il automatise la traduction entre objets Java et tables MySQL. Gain de temps (pas de requêtes SQL à écrire), sécurité (contre les injections SQL) et portabilité (changement de base facilité).

**7. Calcul du CA via la délégation (plutôt que surcharge) :**  
La classe `GestionTournee` calcule le total en itérant sur les objets `Visite`, et appelle la méthode `montantAFacturer()` de chaque `Visite`, qui elle-même multiplie ses actes par le prix du `TypePrestation`.

**8. Rôle de @BeforeEach dans JUnit 5 :**  
Cette annotation exécute la méthode avant chaque test. Elle sert à initialiser un jeu de données "propre" (création d'objets vierges) pour garantir l'indépendance des tests.

**9. Test avec zéro acte :**  
Il valide un cas "limite" pour s'assurer que l'application ne plante pas et que la logique de facturation fonctionne correctement (le CA ne doit pas augmenter si aucun acte n'est facturé).

**10. Communication JavaFX / ApiService :**  
Le contrôleur JavaFX capte les actions de l'utilisateur et appelle `ApiService`. Celui-ci effectue la requête réseau, convertit la réponse (ex: JSON) en objets Java, puis les retourne au contrôleur qui met à jour l'interface.


---
CA TANAVONG SIO 2 SLAM
