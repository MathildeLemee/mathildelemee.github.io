---
layout: post
title: Premier pas avec Firebase
comments: true
---

Firebase est est une plateforme de développement web et mobile. Avec SoFizz, c’est l’outil qui nous permet de gérer et stocker les données des utilisateurs et toute la partie temps réel de l’application (le chat, les notifs, la mise à jours des sorties ...). A l’origine, Firebase n’était qu’une base de données temps réel, qui offrait une API pour conserver et synchroniser des données entre de multiples clients. Désormais, il y a également un service d’authentification, de stockage, de push notifications ….
Cet article est le premier d'une suite et présente les basiques. Le suivant sera basé sur la sécurité et l'authentification Facebook avec ionic, le troisième sur la mise en place d'une architecture CQRS event based / serverless avec l'utilisation des Google Cloud Functions (so hype ;) ).

Les fonctionnalités de Firebase sont découpées en 3 briques : 
### Develop
* RealTime Database
* Authentification
* Cloud Messaging
* Storage 
* Remote Config
* Test Lab 
* Crash Reporting

### Grow
* Notification
* Dynamic Links
* Invites
* Adwords

### Earn
* AdMobs

Le développement de Firebase va très vite, la plupart de ses services n’existaient pas quand on a définit l’architecture du projet. On utilise actuellement  [Branch.io](http://branch.io) pour les dynamic links, [ionic push](https://docs.ionic.io/services/push/) pour les notifications, [cloudinary](http://www.cloudinary.com) pour la gestion des images, Firebase pour la base de données temps réel et l’authentification Facebook dans l’app. L’idée est de migrer petit à petit les services. De l’autre coté, ionic a a peu près le meme sens de développement et propose de plus en plus de features annexes.
Dans un premier temps, pour les premiers mvp, nous n'avions que l'appli ionic avec angularjs qui stockait et lisait directement dans firebase, et aucune couche intermédiaire.
## Firebase, c'est quoi ?

Au niveau de l'API principale, c'est un outil qui nous offre 3 services.
* firebase.auth() - Authentication
* firebase.storage() - Cloud Storage
* firebase.database() - Realtime Database

### Firebase Realtime Database
Nous allons nous intéresser au fonctionnement de la base de données temps réel. Le stockage est très simple, un arbre json dans lequel on va pousser des attributs clé-valeurs. Les fonctionnalitées proposées par Firebase sont très simple, pas de recherche, d'aggrégation ...
Si je veux stocker les informations de profil, de settings ainsi que le nombre de message qu'un utilisateur a à lire, cela pourrait être représenté sur Firebase par les données suivantes :

```javascript
{profil :
  {
      keyUser1 : {firstname : Tom, age : 25, geoloc : {lon : 49.5,lat:2.55}},
      keyUser2 : {firstname : Mat, age : 30, geoloc : {lon : 49.2,lat:2.57}},
      keyUser3 : {firstname : May, age : 26, geoloc : {lon : 49.1,lat:2.59}}
  },
 settings :
  {keyUser1 : {gps : true,notifMail:true,notifPush:false},
   keyUser2 : {gps : false,notifMail:true,notifPush:true}
   keyUser3 : {gps : true,notifMail:false,notifPush:true}
  },
 messageToRead :
   {keyUser3 : 12},
 }
 ```

#### Ajout

Pour écrire sur Firebase, il suffit d'indiquer où doivent être placées les valeurs.

Par exemple, pour ajouter un 4ème membre :
```javascript
firebase.database().ref('profil/keyUser4' ).set({
    firstname: 'Emilie',
    age: 32,
    geoloc : {lon : 49.1,lat:3,07}
  });
```

#### Mise à jour
Il est également possible de ne mettre à jour qu'une partie d'un enregistrement en utilisant update au lieu de set.
Par exemple, si je veux modifier le nom et l'age de keyUser2, il me suffit de faire un update sur ces 2 champs, qui seront alors modifié, pendant que la geoloc sera conservée.
```javascript
firebase.database().ref('profil/keyUser2' ).update({
    firstname: 'EmilieJolie',
    age: 33
   });
```

#### Suppression

De même, pour supprimer un enregistrement, il suffit d'appeler remove.
```javascript
firebase.database().ref('profil/keyUser2' ).remove();
```

### Best Practices d'écritures

Au niveau des best practices, comme on le retrouve souvent dans ce genre de base de données, il est important de ne pas trop imbriquer les données pour pouvoir les récupérer facilement et ne pas récupérer des données inutiles qui encombrent le réseau.
Ainsi, il vaut mieux multiplier les clés d'un niveau supérieur, comme c'est le cas avec profil et settings. Le settings etant lié à l'utilisateur tout comme le profil, il aurait été facile de faire :
```javascript
{profil :
  {keyUser1 : {firstname : Tom, age : 25, geoloc : {lon : 49.5,lat:2.55},settings : {gps : true,notifMail:true,notifPush:false}},
   keyUser2 : {firstname : Mat, age : 30, geoloc : {lon : 49.2,lat:2.57},settings : {gps : true,notifMail:true,notifPush:true}},
   keyUser3 : {firstname : May, age : 26, geoloc : {lon : 49.1,lat:2.59},settings : {gps : true,notifMail:false,notifPush:true}}},
 messageToRead :
   {keyUser3 : 12},
 }
 ```

Sauf que le profil est chargé à chaque connection de l'utilisateur, alors que les settings ne le sont que quand l'utilisateur décide de modifier ses paramètres.
Ainsi, si je fais le choix d'avoir mes données imbriquées, a chaque connection je charge le profil et les settings alors que dans la première optique, je charge le profil et les settings ne sont chargées qu'éventuellement.

### Lecture

Il y a deux moyens de récupérer les données en lecture. Avec 'on' ou avec 'once'.

#### On
Avec on, la méthode permet de récupérer les données
via la méthode val sur l'objet transmis au callback. C'est un listener, c'est à dire qu'à chaque modification du profil de l'utilisateur (ajout d'un nouveau fils, mise à jour d'un descendant ...), la méthode indiquée sera appelée.
Ainsi, il est important d'attacher les listeners sur les plus petites entitées possibles, comme vu dans les best practices précedemment, pour garantir que la taille des snapshots récupérés soient les plus petits.

```javascript
var userId = firebase.auth().currentUser.uid;
var onValue = function(snapshot) {
                var firstname = snapshot.val().firstname;
                // ...
              }

return firebase.database().ref('/profil/' + userId).on('value',onValue);
```

Il est possible de détacher le callback en utilisant off().
```javascript
var userId = firebase.auth().currentUser.uid;
var onValue = function(snapshot) {
                var firstname = snapshot.val().firstname;
                // ...
              }

return firebase.database().ref('/profil/' + userId).off('value',onValue);
```

#### Once
Si l'on ne souhaite pas avoir ce méchanisme de listerner mais juste récupérer les valeurs, il suffit d'utiliser 'once' à la place de 'on'.

```javascript
var userId = firebase.auth().currentUser.uid;
return firebase.database().ref('/profil/' + userId).once('value').then(function(snapshot) {
  var firstname = snapshot.val().firstname;
  // ...
});
```

#### eventType

Dans les deux exemples suivants, nous récupérions l'ensemble de l'objet via l'event type 'value'. Il existe différents event type
 "value", "child_added", "child_changed", "child_removed", or "child_moved." Ainsi, si je veux lancer un callback lors de la suppression d'un enfant, il suffit de passer 'child_removed' comme event type.

  ```javascript
  var userId = firebase.auth().currentUser.uid;
  return firebase.database().ref('/profil/' + userId).on('child_removed').then(function(snapshot) {
    // ...
  });
  ```




## Pour finir

Nous avons parcouru une bonne partie de l'API de firebase. Il nous manque la possibilité de ne pouvoir récupérer qu'un subset des données (avec startAt et endAt) et la possibilité de faire des transactions. L'API de firebase est très simple à prendre en main, le plus compliqué restant de bien architecturer ses données.
Dans l'article suivant, nous verrons comment utiliser l'authentification de Firebase et la possibilité de définir de manière fine des droits en lecture et écriture sur chacune des clés.

{% if page.comments %}
<div id="disqus_thread"></div>
<script>

var disqus_config = function () {
this.page.url = {{ page.url }};  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = '{{ site.disqus_shortname }}'; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};

(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://exploratorycoding.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>

{% endif %}
