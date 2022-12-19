Bonjour tout le monde, et merci d'être venus ce soir.

Pour commencer, une petite question.

Qui a déjà programmé en Python ?

...

Ok, je me suis pas trompé de Meetup ;)

Qui a déjà programmé en Javascript ?

Qui a déja utilisé un langage "fonctionnel" ? (type Lisp, Haskell, OCaml, Scala...)

...

Un peu moins, OK.

Ce soir on va voir comment fonctionne la gestion des erreur en Python, aller de piocher dans ce qui se fait ailleurs, et voir si ça peut nous être utile en Python. Et au passage on fera un peu de compote.

Deux mots me concernant : je m'appelle Samuel, je travaille chez Lowit, on modélise le comportement thermique des bâtiments pour mettre en oeuvre la transition énergétique.

---

En tant que développeurs, on essaie de faire des programmes qui marchent et qu'on puisse modifier facilement. Ce ne sont pas les seuls critères mais c'est souvent assez haut parmis nos priorités.

Qui parmis vous essaie d'écrire du code maintenable ?

Python est plutôt sympa sur ce point. D'ailleurs, dans la philosophie de Python, il y a:

> Explicit is better than implicit.

C'est à dire qu'on préfère quand visible et compréhensible que quand elles sont cachées et "magiques". C'est plus maintenanble.

Dans le Zen de Python, on trouve aussi:

> Errors should never pass silently.
>
> Unless explicitly silenced.

C'est à dire que lorsque quelque d'incorrect se produit, c'est plus facile à corriger si la langage nous le dit haut et fort que si c'et masqué.

Et ça, c'est géré avec des exceptions et des try/except. Par exemple:

```python
def calculer_imc(taille, poids):
    return poids / (taille * taille)
```

Quand on utilise cette fonction dans le premier cas, tout se passe bien. Dans le deuxième cas, Python nous dit explicitement qu'il y a une erreur dans la division.

```python
>>> calculer_imc(1.93, 74) # OK
19.86630513570834
>>> calculer_imc(0, 74)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 2, in calculer_imc
ZeroDivisionError: division by zero
```

Ca marche tellement bien que même si on enterre cette fonction dans les entrailles de notre programme, l'exception remonte tout en haut:

```python
def traiter_requete_http(requete):
    id = int(requete.payload.id)
    imc = calculer_imc_dun_utilisateur(id)
    return {"Result" : imc}, 200

def calculer_imc_dun_utilisateur(id_utilisateur):
    utilisatuer = db.trouver_un_utilisateur_par_id(id_utilisateur)
    return calculer_imc(utilisatuer)

def calculer_imc_dune_personne(personne):
    return calculer_imc(personne.taille, personne.poids)

def calculer_imc(taille, poids):
    return poids / (taille * taille)
```

Quand on utilise ça avec l'identifiant d'un utilisateur dont le taille est à zéro, l'erreur remonte tout en haut, et le programme s'arrête avec un message d'erreur. Parfait !

```text
traiter_requete_http
    -> calculer_imc_dun_utilisateur
        -> calculer_imc_dune_personne
            -> calculer_imc
```

Pour empêcher le programme de s'arrêter brutalement, par exemple si on programme un serveur et qu'on souhaite qu'il continue de fonctionner même si certaines données sont incohérentes, il suffit de penser à utiliser un try/except.

```python
def traiter_requete_http(requete):
    id = int(requete.payload.id)
    try:
        calculer_imc_dun_utilisateur(id)
        return {"Result" : imc}, 200
    except ZeroDivsionError as e:
        log("Une erreur s'est produite lors du traitement de cette requête.", e)
        return {"Result" : "Error"}, 500
```

Alors, je dois vous avouer un truc : je suis un peu bête. Quand "il faut juste penser à" faire quelque chose, assez souvent j'oublie. Même si c'est écrit en gros, en gras, en rouge et en clignotant dans la documentation, j'oublie. Et puis de toute façon, je lis pas la doc, alors...

Bon, vous allez me dire "Oui mais, Samuel, ton exemple est pourri, il pourrait être mieux écrit." C'est vrai. Mais dans la vraie vie, en dehors de présentation, on peut avoir du code plus complexe. Par exemple chez Lowit on modélise des parcs de bâtiments pour calculer leur consommation d'énergie. Chaque bâtiment est divisé en plusieurs parties et chaque partie doit être associée à une profil d'utilisation, etc etc.

Tant que les modélisateurs sont en train de modéliser un bâtiment, toutes les infos ne sont pas encore définies. On ne peut pas encore calculer la consommation de ce bâtiment.

```python
def un_bout_de_code_au_plus_profond_des_entrailles(args):
    if args.schroumpf is None:
        raise ErreurPasDeSchtroumpf("Sauve qui peut !!!")
    
    return args.toto / args.schroumpf
```

Si on essaie de calculer la consommation, un bout de code au plus profond des entrailles de Lowit lance une exception. On espère qu'elle sera attrapée plus tard. Et quand on utilise une fonction, on ne sait jamais vraiment quand elle peut exploser à moins de lire le code source. Par exemple ce petit bout de code innocent:

```python
from flask import Flask, request
import json

app = Flask(__name__)

@app.route('/json-example', methods=['POST'])
def json_example():
    data = request.get_json()
    result = save_foo_bar_baz(data)
    return json.dumps(result)
```

Et bien si la requête est malformée, "get_json" explose. Si les données sont invalides ou bien qu'il y a une problème de connexion à la base de donnée, "save_foo_bar_baz" explose aussi.

IMAGE SIEGE EJECTABLE !!!

On pourrait appeler ça la "Programmation Orientée Siège Ejectable": on tire sur la manette et on espère avoir un parachute pour retomber doucement.

Ce style de programme est assez facile à modifier, mais aussi assez facile à casser par inadvertance.

En vrai, ça marche quand même assez bien.

Mais peut-on faire mieux ?

Les gens qui font des langages purement fonctionnels ont imanginé un concept intéressant: la programmation orienté train ("Railway Oriented Programming" en VO, ROP pour les intimes). C'est une façon de gérer les erreurs d'une façon plus explicite.

En programmation fonctionnelle : une fonction pure, c'est une fonction qui prend une entrée et qui retourne un résultat.

IMAGE function PommeBanane

```python
def inverse(x):
    return 1/x
```

L'idée de la programmation tchou tchou, c'est de dire que le resultat d'une fonction peut être soit une valeur, soit une erreur

IMAGE fonction switch

```python
def inverse(x):
    if x != 0:
        return 1/x
    else:
        return "Erreur"
```

Pour distinguer plus facilement la valeur d'une erreur, on peut les emballer :

```python
@dataclass
class Ok:
    value: Any

@dataclass
class Error:
    error: Any

def inverse(x):
    if x == 0:
        return Error("Erreur")
    else:
        return Ok(1/x)
```

Un truc sympa c'est de composer deux fonctions ensemble. Par exemple avec une fonction Pomme Banane et une fonction Banane Cerise, on peut faire une fonction Pomme Cerise

IMAGE fonctions pomme banane ET banane cerise

IMAGE fonction pomme cerise.

```python
def inverse(x):
    return 1/x

def ajouter_deux(x):
    if x > 10:
        raise Exception("C'est trop grand !")
    return x + 2

def toto(x):
    """Retourne 2 + 1/x"""

    return ajouter_deux(inverse(x))
```

Mais comment composer deux fonctions qui retournent des erreurs ?

IMAGE fonctions swith PAS composées

Dans le cas où tout se passe bien, c'est facile, mais dans le cas où il y a une erreur, on doit contourner la fonctionnement normal.

On doit ajouter un segment pour que les fonctions prennent en entrée soit une valeur normale, soit l'erreur.

IMAGE fonctions Switch Composées

```python
def inverse(x_ou_erreur):
    match x_ou_erreur:
        case Ok(x)
            if x == 0:
                return Error("Erreur")
            else:
                return Ok(1/x)
        case Error(erreur):
            return Error(erreur)

def ajouter_deux(x_ou_erreur):
    match x_ou_erreur:
        case Ok(x)
            if x > 10:
                return Error("C'est trop grand !")
            return x + 2
        case Error(erreur):
            return Error(erreur)

def toto(x):
    """Retourne 2 + 1/x ou une erreur."""

    return ajouter_deux(inverse(x))
```

C'est lourd. Ca se répète.

IMAGE adaptateur pour switch

Ce qui serait bien ce serait d'avoir un adaptateur qui nous permette de une fonction a un  rail en entrée en une fonction à deux rails en entrée.

Qu'est-ce que ferait cet adaptateur ? Dans le cas où le paramètre est une valeur, ça passe la valeur à la fonction. Sinon, quand le paramètre est une erreur, ça transmet l'erreur sans rien faire.

Le plus simple, c'est d'ajouter directement cette logique au classes Ok et Error qu'on a défini plus tôt:

```python
@dataclass
class Ok:
    value: Any

    def then(self, f):
        return f(self.value)

@dataclass
class Error:
    error: Any

    def then(self, f):
        return self
```

[Petit moment pour expliquer]

```python
def inverse(x_ou_erreur):
    if x == 0:
        return Error("Erreur")
    else:
        return Ok(1/x)

def ajouter_deux(x_ou_erreur):
    if x > 10:
        return Error("C'est trop grand !")
    return x + 2

def toto(x):
    """Retourne 2 + 1/x ou une erreur."""

    return inverse(x).then(ajouter_deux)
```
