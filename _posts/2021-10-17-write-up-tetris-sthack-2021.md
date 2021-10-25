---
title: "Write up - Tetris - Sthack 2021"
---
Dans cet article, je vais vous présenter le write up d’une épreuve web/programmation que j'ai créé pour le CTF de la Sthack 2021.

# Tour d'horizon du challenge
Lorsque l'on arrive dans le challenge, on a deux modes possibles :
- Le mode "challenge" qui contient le flag
- Et le mode "fun" a été ajouté pour que les participants puissent jouer si certains s'ennuient ou veulent se libérer un peu l'esprit avec un petit Tetris.

![home](/assets/images/sthack_2021_tetris/home.png)

Dans ces deux modes, les règles sont les mêmes que le Tetris classique. 
Il y a donc plusieurs niveaux, qui permettent d'obtenir plus de points qu'une pièce est posée et lorsqu'une ou plusieurs lignes sont détruites. Si plusieurs lignes sont détruites à la fois, ça permet d'obtenir plus de points d'un coup.
À noter aussi que la vitesse des blocs est accélérée plus, les niveaux avancent.

## Mode Challenge
Le but du mode challenge est de faire le maximum de score en 3 minutes.
En jouant, on se rend compte très vite que le score n'est pas battable en jouant normalement. Ce qui veut dire que Nitnick du 33 a triché, mais comment ? 

![challenge mode](/assets/images/sthack_2021_tetris/mode_challenge.png)

# Trois solutions possibles
La solution que j'avais pensée pour ce challenge était de faire un script qui permet de réaliser ce highscore. Cette solution est présentée dans la Solution 1. 

Mais les équipes participantes ont trouvé deux solutions différentes de résoudre ce challenge. 
- Dans la Solution 2, je vous ai mis le lien vers le write up de la team "ZenSec" qui a réalisé un cracke du challenge.
- Dans la Solution 3, la team "Les pires Hats" utilisé une extension Chrome/Firefox appelée Cetus et qui permet de changer dynamiquement la mémoire du jeu.

# Solution 1 - Programmer un script pour jouer à notre place
## Comment communiquer avec unity
La première étape est de savoir comment communiquer avec le jeu pour pouvoir exécuter des actions dans le jeu.
On sait déjà que Unity WebGL est réalisé en partie en Web assembler. Si on connait un peu la technologie web assembler on sait qu'il y a une communication entre le Javascript et le code compilés.  

Donc, il est possible que Unity permette une communication avec le jeu en Javascript.

Après une petite recherche ~~Google~~ Duckduckgo, on tombe sur la page documentation de Unity [WebGL: Interacting with browser scripting](https://docs.unity3d.com/Manual/webgl-interactingwithbrowserscripting.html "Documentation Unity WebGL" ). 
Dedans il y a le paragraphe intitulé "Calling Unity scripts functions from JavaScript" dans lequel on apprend que l'on peut interagir directement en js avec les méthodes des classes du jeu (les GameObjects) comme ceci :

{% highlight js %}
unityInstance.SendMessage(objectName, methodName, value);
{% endhighlight %}

Avec comme exemple d'utilisation :
{% highlight js %}
unityInstance.SendMessage('MyGameObject', 'MyFunction');
unityInstance.SendMessage('MyGameObject', 'MyFunction', 5);

unityInstance.SendMessage('MyGameObject', 'MyFunction', 'MyString');
{% endhighlight %}

Attention, seules les méthodes publiques sont utilisables !

Reste à savoir si le jeu expose la variable "unityInstance". On cherche dans le code HTML et on trouve une balise <script></script> dans laquelle est déclarée la variable unity_instance.
![unity_instance_variable](/assets/images/sthack_2021_tetris/unity_instance_variable.png)

On fait un essai :

![unity_instance_test](/assets/images/sthack_2021_tetris/unity_instance_test.png)

Cela fonctionne, mais il manque le nom des GameObject pour pouvoir réaliser le script.

## Récupération nom des GameObject
Deux méthodes pour récupérer les noms des GameObject :
### Méthode avec le projet WABT
Le projet WABT est une suite d'outil développé par le projet Webassembly et qui permet en autre de facilité le debbug des binaires Webassembly. 
Le projet est disponible sur github [wabt](https://github.com/WebAssembly/wabt "Le github du projet WABT").

Dans cette suite d'outil, c'est wasm2wat qui va nous être utile. En effet cet outil permet de transcrire le webassembly dans un langage human readable.
{% highlight bash %}
wasm2wat challenge/Build/TetrisChallenge.wasm -o challenge/Build/TetrisChallenge.wat
{% endhighlight %}

Dans le fichier .wat, on finit par trouver GameObject qui nous intéresse :
- GameController
- Spawner
- Holder
- Next

Et en lisant bien le code, on trouve les méthodes associées aux GameObject.
### Utiliser AssetStudio
Dans cette méthode, on va récupérer le nom des méthodes et GameObject à l'aide de l'outil AssetStudio. 
Cet outil uniquement disponible sur Windows permet d'explorer et d'extraire les assets des jeux unity et est disponible sur github [Perfare/AssetStudio](https://github.com/Perfare/AssetStudio "Github assetStudio")

Pour récupérer les noms des GameObject et de leurs méthodes, on faut explorer les métadata du jeu. 
Pour extraire le fichier de metadata dans AssetStudio, il faut tout d'abord récupérer les data du jeu. 
Pour cela, il faut regarder dans l'onglet "Réseau" de la console du navigateur est récupéré le fichier TetrisChallenge.data.

Une fois récupéré, il faut extraire le fichier .data dans AssetStudio en faisant :

![extract_file](/assets/images/sthack_2021_tetris/asset_studio_extract_file.PNG)

Puis, il faut sélectionner le fichier .dat puis sélectionner le dossier de réception

Dans le résultat de l'extraction, le fichier global-medata.dat se situe dans : Il2CppData\Metadata\global-metadata.dat

Une fois le fichier trouver, on fait la commande strings :

Et en cherchant bien, on retrouve le nom des GameObject suivi de leurs méthodes et des variables.

Extrait de la commande strings: 
{% highlight bash %}
GameController
pauseGame
hold
setGameOver
isGameOver
rotate
moveLeft
moveRight
moveDown
drop
getLevel
computeMove
computeTimer
resetGame
gameStopped
startGame
to_add
increaseScore
computeFlag
computeLevel
computeSpeed
increaseLinesDeleted
updateScoreText
updateLevelText
computeScore
{% endhighlight %}

## Réalisation du script
### Test des méthodes dans la console du navigateur
On va essayer de faire spawner le bloc que l'on souhaite avec le GameObject Spawner et la méthode Spawn. En faisant :
{% highlight js %}
unity_instance.SendMessage('Spawner', 'spawn')
{% endhighlight %}

Mais l'erreur suivante apparait :
{% highlight js %}
Failed to call function spawn of class Spawner
Calling function spawn with no parameters but the function requires 1.
{% endhighlight %}

Il faut donc qu'il y ait un paramètre à la fonction pour être utilisé et puis dans le fichier .wat ou dans le global-data, on voit des méthodes qui parle d'index, cela signifie que les blocs ont des index et sont donc appelables, grâce à ça.
On essaye donc :
{% highlight js %}
unity_instance.SendMessage('Spawner', 'spawn', 1)
{% endhighlight %}

Ça fonctionne, mais pas comme on l'aurait souhaité. En effet, le bloc apparait, mais le premier bloc du jeu lui s'immobilise au moment où on l'on fait apparaitre le bloc.

![spawner_problem](/assets/images/sthack_2021_tetris/spawner_problem.png)

Il faut donc essaye autrement. On peut essayer de passer par le Next et faire spawner les prochains blocs comme on le souhaite et mettre le premier bloc dans le hold.
{% highlight js %}
unity_instance.SendMessage('Next', 'spawnNext', 1)
{% endhighlight %}

Ça fonctionne, mais si l'on exécute une seconde fois, on se rend compte que le next fait spawner un autre type de bloc, cela veut dire que la méthode spawnerNext ne prend pas en compte le paramètre que l'on passe.
Donc, là encore on ne peut pas passer par le next pour faire spawn ce que l'on souhaite.

Il reste donc plus qu'un endroit pour faire spawn les blocs que l'on souhaite, c'est par le hold. On essaye la commande suivante :
{% highlight js %}
unity_instance.SendMessage('Holder', 'spawnHolder', 1)
{% endhighlight %}

Cela fonctionne aussi, même en exécutant plusieurs fois la ligne. On peut avoir le contrôle sur le holder.

## Coder le script
L'idée du script est de faire apparaitre le bloc de la barre verticale (qui est l'index 6 après avoir testé les différents index) et de résoudre 4 lignes par 4 lignes histoire d'avoir le maximum de score à la fois. 

Pour cela, on va :
- Reset le holder
- Faire apparaitre la barre
- Puis simuler un hold
- Placer la barre à l'horizontale à gauche x4
- Placer la barre à l'horizontale au milieu x4
- Placer la barre à la verticale à droite
- Placer la barre à la verticale toute à droite

Et le script donne cela :
{% highlight js %}
function spawnholder() {
unity_instance.SendMessage("Holder", "reset");
unity_instance.SendMessage("Holder", "spawnHolder", 6);
}
function hold() {
spawnholder();
unity_instance.SendMessage("GameController", "hold");
}

function leftPosition() {
hold();
unity_instance.SendMessage("GameController", "rotate");
unity_instance.SendMessage("GameController", "moveLeft");
unity_instance.SendMessage("GameController", "moveLeft");
unity_instance.SendMessage("GameController", "moveLeft");
unity_instance.SendMessage("GameController", "drop");
}

function middlePosition() {
hold();
unity_instance.SendMessage("GameController", "rotate");
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "drop");
}

function verticalPosition1() {
hold();
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "drop");
}
function verticalPosition2() {
hold();
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "moveRight");
unity_instance.SendMessage("GameController", "drop");
}

function do_action() {
for(var i = 0; i < 4;i++) {
leftPosition();
middlePosition();
}
verticalPosition1();
verticalPosition2();
}

function beatTheHighScore() {
unity_instance.SendMessage("GameController", "rotate");
for(var i = 0; i < 25; i++) {
try {
do_action();
}
catch (e) {
console.log("Flag found");
}

}
}
beatTheHighScore();
{% endhighlight %}

## Lancer le script
Pour lancer le script dans la console, j'utilise Firefox et son option pour exécuter plusieurs à la fois dans la console. Pour l'activer il faut cliquer ici : 

![firefox_multiline_option](/assets/images/sthack_2021_tetris/firefox_mutliline_option.png)

Une fois le code placer sur la gauche, il reste plus qu'à l'exécuter autant de fois que voulu pour battre le highscore. 
À noter que dans mon script, je fais l'opération 4 par 4, car sinon Firefox plante. 

![flag_found](/assets/images/sthack_2021_tetris/flag_found.png)

Et le flag est : **1Lov3TetrisGame#RIPJonasNeubauer**

PS : Jonas Neubauer a été 7 fois champion du monde de tetris et est décédé au mois de Janvier 2021. Ce flag était un petit hommage à ce champion.
# Solution 2 - Cracker le jeu 
Cette solution a été utilisée par la team "ZenSec" et présentée dans leur write-up disponible [ici](https://zensecctf.github.io/sthack2021/2021-10-19-sthack2021-tetris "Write up craking"), donc n'hésitez pas à jeter un œil.

# Solution 3 - Utiliser l'extension Cetus
C'est la solution qui a été utilisée par la team "Les Pires hat" et présentée dans leur write-up disponible [ici](https://lespireshat.medium.com/sthack-2021-write-up-tetris-ddf27faf6a35 "Write up cetus"), donc n'hésitez pas à jeter un oeil.