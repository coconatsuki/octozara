---
title: "Réduire ses contrôleurs et modèles"
date: 2019-02-03T12:46:12+09:00
type: post
categories: ["code"]
---

## Un problème de responsabilité et de régime

{{< youtube PjV6inAke8k >}}

Prenons une méthode de contrôleur "classique":

```ruby
def update
  # récuperer les paramètres (strong params) (et gérer les erreurs)
  begin
    update_params = params.require(:post).permit(:text)
  rescue ActionController::ParameterMissing => e
    flash[:error] = e.message
    return redirect_to(root_path)
  end

  # trouver le post et gérer le cas d'erreur
  @post = Post.find_by(id: params[:id])
  unless @post
    flash[:error] = "Post not found"
    return redirect_to(root_path)
  end

  # vérifier que le current user a le droit de update et gérer le cas d'erreur
  if @post.author_id != current_user.id
    flash[:error] = "You aren't authorized"
    return redirect_to(root_path)
  end
  # Sauvegarder le texte actuel du post (ça va servir plus tard)
  current_text = @post.text
  # update et vérifier qu'on peut bien sauver et gérer les cas d'erreur
  if @post.update(update_params)
    # récupérer le nouveau texte du post et le nickname du current_user ; on le récupère dans slack en cas d'absence.
    slack_nick = current_user.slack_nick
    unless slack_nick
      slack_nick = $slack_client.users_info(user: current_user.slack_id)
      if slack_nick
        current_user.update(slack_nick)
      end
    end
    unless slack_nick
      flash[:error] = "Can't find the user in slack O_o"
      # ERROR ! Rollback !
      @post.update(text: current_text)
      return redirect_to(root_path)
    end
    text = update_params.text
    # envoyer le message slack pour annoncer l'update
    $slack_client.chat_postMessage(
      channel: '#general',
      text: "@#{slack_nick} haz updated the post##{@post.id} with: #{text}",
      as_user: true
    )
  else
    flash[:error] = "Something went wrong: #{@post.errors.full_messages.join(", ")}"
    return redirect_to(root_path)
  end
end
```

Un lecteur avisé et assidu de mon blog (que vous êtes surement) devrait se dire: “_Mon dieu… brulez ça_”.
Et il aurait raison: Ce contrôleur est incompréhensible et beaucoup trop gros.
Croyez moi, ce genre de choses arrivent dans la vraie vie et ça peut même être bien pire…

L'exercice que nous allons mener à présent est le suivant :

**Lui donner une cure d'amaigrissement par diverses méthodes pour le ramener a une taille raisonnable.**

PS: J'entends déjà les petits malins qui me disent: “_Bah tu peux tout mettre dans les modèles_”: Non™.
PPS: Si vous voulez jouer, trouvez le bug dans la méthode ci-dessus (C'est fun a chercher dans une méthode gigantesque hein ?)

**DISCLAMER: Ce choix d'architecture est le mien et ce post reflète mon opinion.**

## Gérer ses exceptions

Première technique, la gestion des exceptions:

**1: Dans les contrôleurs, il est possible de définir des méthodes à appeler pour rescue une exception.**

La méthode s'appelle [rescue_from](https://api.rubyonrails.org/classes/ActiveSupport/Rescuable/ClassMethods.html)

On va donc retravailler ce "magnifique"

```ruby
begin
  update_params = params.require(:post).permit(:text)
rescue ActionController::ParameterMissing => e
  flash[:error] = e.message
  return redirect_to(root_path)
end
```

Qui va devenir :

Dans la méthode de contrôleur:

```ruby
# rien
```

en dehors de la méthode mais dans le contrôleur:

```ruby
class PostsController < ApplicationController
  rescue_from ActionController::ParameterMissing, with: :bad_parameters

  […]
  private

  # On remarque qu'on a plus besoin de return
  def bad_parameters(exception)
    flash[:error] = e.message
    redirect_to(root_path)
  end

  # le `@update_params ||=` est une technique pour ne pas réexecuter
  # le second bout de la méthode a chaque fois.
  # Il sera calculé la première fois (car @update_params sera nil),
  # puis plus jamais (car @update_params vaudra la bonne valeur).
  def update_params
    @update_params ||= params.require(:post).permit(:text)
  end
end
```

## Les méthodes "exceptionnelles" de ActiveRecord

Savez vous quelle est la différence entre `find_by(id: xxx)` et `find(xxx)` ou `find_by!(id: xxx)` ?
De même entre `update`/`create` et `update!`/`create!` ?

La réponse est assez simple, les secondes feront remonter une exception et arrêteront donc l'exécution du code en cours.

**2: Pensez à utiliser la version "avec exception" des méthodes à moins que vous n'ayez pas besoin de gérer l'échec.**

Nous l'implémentons dans notre méthode de contrôleur (couplé à la règle 1) :

```ruby
# trouver le post et gérer le cas d'erreur
@post = Post.find_by(id: params[:id])
unless @post
  flash[:error] = "Post not found"
  return redirect_to(root_path)
end
```

devient

```ruby
class PostsController < ApplicationController
  rescue_from ActiveRecord::RecordNotFound, with: :record_not_found
  […]

  def update
    […]
    @post = Post.find(params[:id])
    […]
  end

  private
  […]

  # Encore une fois pas de return
  def record_not_found
    flash[:error] = "Post not found"
    redirect_to(root_path)
  end

  […]
end
```

De même

```ruby
if @post.update(update_params)
  […]
else
  flash[:error] = "Something went wrong: #{@post.errors.full_messages.join(", ")}"
  return redirect_to(root_path)
end
```

devient

```ruby
class PostsController < ApplicationController
  rescue_from ActiveRecord::RecordInvalid, with: :record_invalid
  […]
  def update
    […]
    @post.update!(update_params)
    […]
  end

  private
  […]
  def record_invalid(e)
    # L'exception contient le post qui a échoué
    flash[:error] = "Something went wrong: #{e.record.errors.full_messages.join(", ")}"
    redirect_to(root_path)
  end
  […]
end
```

### Bonus

On pourrait même externaliser ces comportements de `rescue` d'exceptions dans notre `ApplicationController`.
En effet, il y a de grande chances que la majeure partie de notre application se comporte comme ça.

Ici on a donc:

{{< filename "app/controllers/application_controller.rb" >}}

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActionController::ParameterMissing, with: :bad_parameters
  rescue_from ActiveRecord::RecordNotFound, with: :record_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :record_invalid
  […]

  private

  def record_invalid(exception)
    flash[:error] = "Something went wrong: #{exception.record.errors.full_messages.join(", ")}"
    redirect_to(root_path)
  end

  def record_not_found
    # Il est probablement possible d'avoir le nom du model depuis l'exception.
    flash[:error] = "Record not found"
    redirect_to(root_path)
  end

  def bad_parameters(exception)
    flash[:error] = exception.message
    redirect_to(root_path)
  end
end
```

et

{{< filename "app/controllers/posts_controller.rb"  >}}

```ruby
class PostsController < ApplicationController
  def update
    # trouver le post
    @post = Post.find(params[:id])

    # verifier que le current user à le droit d'update et gérer le cas d'erreur
    if @post.author_id != current_user.id
      flash[:error] = "You aren't authorized"
      return redirect_to(root_path)
    end

    # Sauvegarder le texte actuel du post (ça va servir plus tard)
    current_text = @post.text

    # update
    @post.update!(update_params)

    # récuperer le nouveau text du post et le nickname du current user (si on a pas son nick aller le chercher dans slack)
    slack_nick = current_user.slack_nick
    unless slack_nick
      slack_nick = $slack_client.users_info(user: current_user.slack_id)
      if slack_nick
        current_user.update(slack_nick)
      end
    end
    unless slack_nick
      flash[:error] = "Can't find the user in slack O_o"
      # ERROR ! Rollback !
      @post.update(text: current_text)
      return redirect_to(root_path)
    end
    text = update_params.text
    # envoyer le message slack pour annoncer l'update
    $slack_client.chat_postMessage(
      channel: '#general',
      text: "@#{slack_nick} haz updated the post##{@post.id} with: #{text}",
      as_user: true
    )
  end

  private

  def update_params
    @update_params ||= params.require(:post).permit(:text)
  end
end
```

Ce n'est pas parfait mais c'est déjà mieux. On va maintenant attaquer le découpage avec des services (un seul ici).

## Les services

Les services représentent pour moi une façon de se "protéger" et rendre transparent l'appel a un service tiers (ici Slack) pour le reste de notre application.
Grâce aux services, il est possible de :

- Ne pas avoir de variable globale ou autres trucs étranges;
- Ne pas obliger le reste de notre code a savoir comment marche la gem/l'API qui cache;
- Utiliser des valeurs/comportements par défaut (par exemple, on peut décider que par defaut `as_user` sera toujours `true`);
- Gérer les exceptions particulières du service tiers a un seul endroit (non nécessaire ici)

On crée donc un dossier `app/services`. Dedans on va créer un fichier ruby pour accueillir notre service slack `app/services/slack_service.rb`

En général (et là c'est plus du cas par cas), on aura besoin d'initialiser nos service qu'une seule fois dans toute notre application.
On peut le faire dans un `initializer` au lancement de notre application puis le stocker dans une variable globale et l'utiliser comme ça ensuite.
Ou on peut décider de faire de notre service un [singleton](https://ruby-doc.org/stdlib-2.6/libdoc/singleton/rdoc/Singleton.html).
Un singleton assure qu'il y aura toujours dans notre application au plus 1 instance de la classe.

**3: Protégez vous des services externes en les englobant dans des Services**

{{< filename "app/services/slack_service.rb" >}}

```ruby
require 'singleton' # Ceci pourrait être fait ailleurs mais bon…

class SlackService
  include Singleton

  def initialize
    Slack.configure do |config|
      config.token = ENV['SLACK_API_TOKEN']
    end

    @client = Slack::Web::Client.new
  end
end
```

On pourra ensuite y accéder de la façon suivante: `SlackService.instance` (le `initialize` ne sera appelé que la première fois).

On va maintenant extraire du contrôleur les différentes méthodes qui faisaient appel a Slack (et se rendre compte qu'il y avait un bug)

```ruby
class SlackService
  class SlackError < StandardError; end
  […]

  def fetch_username_from_id(id)
    # Le bug était ici, on ne récupère pas vraiment le username mais un objet entier
    # C'est facile de le voir maintenant qu'on a presque plus rien dans la méthode ;)
    json = @client.users_info(user: id)
    if json.ok
      json.user.name
    else
      raise SlackError, json.error
    end
  end

  def post_message(text, channel: "#general", as_user: true)
    @client.chat_postMessage(
      channel: channel,
      text: text,
      as_user: as_user
    )
  end
end
```

Ce qui donnera dans notre controller

```ruby
class PostsController < ApplicationController
  rescue_from SlackService::SlackError, with: :slack_error

  def update
    […]

    # On va passer a une variable d'instance
    @current_text = @post.text

    […]

    slack_nick = current_user.slack_nick
    unless slack_nick
      slack_nick = SlackService.instance.fetch_username_from_id(current_user.slack_id)
      current_user.update(slack_nick)
    end
    # envoyer le message slack pour annoncer l'update
    SlackService.instance.post_message("@#{slack_nick} haz updated the post##{@post.id} with: #{update_params.text}")
  end

  private

  def slack_error(e)
    flash[:error] = "Something went wrong in slack O_o"
    # Vous trouvez ça moche ? C'est normal.
    @post.update(text: @current_text) if @current_text
    redirect_to(root_path)
  end

  […]
end
```

Discutons un peu de la décision de rollback dans le cas d'une erreur.
À mon avis dans un cas comme ça, si Slack décide de ne pas marche on ne devrait pas rollback.
Ce code ci est là pour l'exemple.

### Bonus: Les transactions

On pourrait laisser notre base de données gérer toute seule le rollback avec une [transaction](https://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html) SQL:
Tout ce qui va arriver dans ce block doit bien se passer sinon revient dans l'état dans lequel tu étais avant (un peu comme une sauvegarde).

Dans le cas présent ça donnerait:

{{< filename "app/controllers/application_controller.rb" >}}

```ruby
class ApplicationController < ActionController::Base
  rescue_from SlackService::SlackError, with: :slack_error

  […]

  private

  # Maintenant que cette action est plus générale, on peut l'envoyer dans l'ApplicationController
  def slack_error(e)
    flash[:error] = "Something went wrong in slack O_o"
    redirect_to(root_path)
  end

  […]
end
```

{{< filename "app/controllers/posts_controller.rb" >}}

```ruby
class PostsController < ApplicationController
  def update
    # trouver le post
    @post = Post.find(params[:id])

    # vérifier que le current user a le droit de update et gérer le cas d'erreur
    if @post.author_id != current_user.id
      flash[:error] = "You aren't authorized"
      return redirect_to(root_path)
    end

    # Si la moindre exception arrive là dedans, revient en arrière.
    @post.transaction do
      @post.update!(update_params)

      slack_nick = current_user.slack_nick
      unless slack_nick
        slack_nick = SlackService.instance.fetch_username_from_id(current_user.slack_id)
        current_user.update(slack_nick)
      end
      SlackService.instance.post_message("@#{slack_nick} haz updated the post##{@post.id} with: #{update_params.text}")
    end
  end
end
```

## La gestion des permissions

Pour gérer ses permissions plus facilement, on va passer par la gem [pundit](https://github.com/varvet/pundit).
Elle nous permet d'extraire nos gestions de permissions dans des "policies".
Je ne vais pas m'attarder sur pundit, ça fera l'objet d'un futur article.

**4: Vos permissions devraient être dans des classes responsables exclusivement de cela.**

En bref:

{{< filename "app/policies/post_policy.rb">}}

```ruby
class PostPolicy < ApplicationPolicy
  def update?
    record.author_id == user.id
  end
end
```

{{< filename "app/controllers/application_controller.rb">}}

```ruby
class ApplicationController < ActionController::Base
  include Pundit
  rescue_from Pundit::NotAuthorizedError, with: :not_authorized_error

  […]

  private
  […]

  def not_authorized_error
    flash[:error] = "You aren't authorized"
    redirect_to(root_path)
  end
end
```

{{< filename "app/controllers/posts_controller.rb">}}

```ruby
class PostsController < ApplicationController
  def update
    # trouver le post
    @post = Post.find(params[:id])

    # verifier que le current user à le droit de update
    authorize @post

    # Si la moindre exception arrive là dedans, revient en arrière.
    @post.transaction do
      @post.update!(update_params)

      slack_nick = current_user.slack_nick
      unless slack_nick
        slack_nick = SlackService.instance.fetch_username_from_id(current_user.slack_id)
        # On remarquera qu'on avait jamais check si le user pouvait effectivement s'update…
        # Encore un bug qu'on peut maintenant voir parceque c'est petit.
        current_user.update!(slack_nick)
      end
      SlackService.instance.post_message("@#{slack_nick} haz updated the post##{@post.id} with: #{update_params.text}")
    end
  end
end
```

## Les commandes (interactors)

A ce stade, nous avons déjà bien avancé.
Il reste néanmoins toute cette gestion de Slack qui parait bien complexe.

Le push dans slack pourrait aller dans un `after_save` dans le model `Post` mais on se retrouverait toujours à avoir du code pas trop à sa place.
En effet, il n'appartient pas à mon modèle d'être responsable d'interactions avec Slack.

Arrive ici la notion de "couche métier" qui se placera entre vos modèles et vos controllers pour empaqueter les règles propres à votre business.
Vos modèles auront comme unique responsabilité de savoir comment s'interfacer avec la database.
Votre controller sera responsable de décrypter/vérifier les params, vérifier les permissions simples et lancer les actions.
Votre couche intermédiaire fera le reste.

Pour ce faire, j'aime beaucoup utiliser la gem [interactor](https://github.com/collectiveidea/interactor).
Qui n'est à mon sens qu'un wrapper pratique autour du design pattern [Command](<https://fr.wikipedia.org/wiki/Commande_(patron_de_conception)>).

L'idée de ce pattern est de dire: “_Il permet de séparer complètement le code initiateur de l'action, du code de l'action elle-même._”

Un interactor ressemble à ceci:

```ruby
class NomDeLActionExplicite
  include Interactor

  def call
    # Fait l'action
  end
end
```

Une classe avec une seule méthode publique: `call`.
Il peut y avoir plusieurs sous fonctions `private` quand l'action en a besoin.
Il vaut mieux néanmoins garder des actions les plus unitaires possibles (ça va permettre de les réutiliser plus tard ailleurs dans notre code un peu comme des Lego).

On appelle l'action de la manière suivante n'importe où dans notre code: `NomDeLActionExplicite.call({context})`.
Le context est un "hash" qui est passé à l'action quand on l'appelle et sera retourné par l'action.

Par exemple:

```ruby
# Oui, cette action est globalement inutile
class CapitalizeText
  include Interactor

  def call
    context.capitalized_text = context.text.capitalize
  end
end
```

```ruby
result = CapitalizeText.call({text: "zaratan"})
result.capitalized_text # => "Zaratan"
```

**5: Vos règles métier n'ont rien à faire ni dans vos contrôleurs et ni dans vos modèles.**

Ici on peut définir deux actions pour gérer slack:

```ruby
class FetchSlackUsername
  include Interactor

  def call
    slack_nick = context.user.slack_nick
    unless slack_nick
      slack_nick = SlackService.instance.fetch_username_from_id(context.user.slack_id)
      context.user.update!(slack_nick)
    end
    context.slack_nick = slack_nick
  end
end
```

```ruby
class SendPostEditSlackMessage
  include Interactor

  def call
    SlackService.instance.post_message("@#{context.slack_nick} haz updated the post##{context.post.id} with: #{context.post.text}")
  end
end
```

Pour ensuite change notre controller en:

```ruby
class PostsController < ApplicationController
  def update
    # trouver le post
    @post = Post.find(params[:id])

    # verifier que le current user à le droit de update
    authorize @post

    # Si la moindre exception arrive là dedans, revient en arrière.
    @post.transaction do
      @post.update!(update_params)

      context = FetchSlackUsername.call(user: current_user)
      SendPostEditSlackMessage.call(slack_nick: context.slack_nick, post: @post)
    end
  end
end
```

## Un problème de besoin et sortie

Un des gros défaut des interactors est qu'il n'est pas très facile de savoir ce que chacun s'attend à recevoir dans son context ni ce qu'il va y ajouter.

Pour résoudre ce problème, on peut utiliser un "contrat" dans chacun de nos interactors.
On peut le faire en définissant dans chacun un `before` et un `after` [hook](https://github.com/collectiveidea/interactor#hooks) qui vérifierait ce contrat.

On peut aussi utiliser [interactor-contracts](https://github.com/michaelherold/interactor-contracts) qui est une gem qui simplifie leurs définitions.

**6: Définissez clairement ce qu'attend et ce que va faire chaque interactor.**
(Vous me remercierez quand vous en aurez plus de 100 dans un projet.)

```ruby
class FetchSlackUsername
  include Interactor
  include Interactor::Contracts

  expects do
    required(:user).filled
  end

  assures do
    required(:slack_nick).filled
  end

  # On définit les conséquences d'un contrat non respecté.
  # context.fail! va envoyer une exception.
  # Le comportement de base des interactor est de cacher cette exception et de juste marquer le context final comme "failed".
  on_breach do |breaches|
    context.fail!(errors: breaches.flat_map(&:messages), breaches: breaches.flat_map(&:property))
  end

  def call
    slack_nick = context.user.slack_nick
    unless slack_nick
      slack_nick = SlackService.instance.fetch_username_from_id(context.user.slack_id)
      context.user.update!(slack_nick)
    end
    context.slack_nick = slack_nick
  end
end
```

```ruby
class SendPostEditSlackMessage
  include Interactor
  include Interactor::Contracts

  expects do
    required(:slack_nick).filled
    required(:post).filled
    end

  assures do
    # Je laisse souvent ce block même vide comme ça je sais que rien ne sera ajouté.
  end

  on_breach do |breaches|
    context.fail!(errors: breaches.flat_map(&:messages), breaches: breaches.flat_map(&:property))
  end

  def call
    SlackService.instance.post_message("@#{context.slack_nick} haz updated the post##{context.post.id} with: #{context.post.text}")
  end
end
```

Comme précisé en commentaire les interactors cachent les exceptions soulevées en cas de fail cf [doc](https://github.com/collectiveidea/interactor#dealing-with-failure)

Néanmoins, il possible de laisser cette exception arriver en faisant: `.call!` à la place de `.call` — Ça devrait vous rapeller quelque chose ;).

```ruby
class PostsController < ApplicationController
  def update
    # trouver le post
    @post = Post.find(params[:id])

    # verifier que le current user à le droit de update
    authorize @post

    # Si la moindre exception arrive là dedans, revient en arrière.
    @post.transaction do
      @post.update!(update_params)

      context = FetchSlackUsername.call!(user: current_user)
      SendPostEditSlackMessage.call!(slack_nick: context.slack_nick, post: @post)
    end
  end
end
```

En cas de rupture du contrat, une exception va donc arriver. On va faire en sorte de gérer cette exception.

```ruby
class ApplicationController < ActionController::Base
  rescue_from Interactor::Failure, with: :interactor_failure
  […]

  private
  […]

  def interactor_failure
    flash[:error] = exception.context.errors.join(", ")
    redirect_to(root)
  end
end
```

### Bonus: Y a du code dupliqué. On peut DRY ça.

Définissons une classe parente aux interactors pour obtenir un comportement par défaut:

```ruby
class ApplicationInteractor
  # Cette sorcellerie est là pour faire comme si on avait écrit ça directement dans la classe fille.
  def self.inherited(base)
    base.instance_exec do
      include Interactor
      include Interactor::Contracts

      on_breach do |breaches|
        context.fail!(errors: breaches.flat_map(&:messages), breaches: breaches.flat_map(&:property))
      end
    end
  end
end
```

On a donc:

```ruby
class FetchSlackUsername < ApplicationInteractor
  expects do
    required(:user).filled
  end

  assures do
    required(:slack_nick).filled
  end

  def call
    slack_nick = context.user.slack_nick
    unless slack_nick
      slack_nick = SlackService.instance.fetch_username_from_id(context.user.slack_id)
      context.user.update!(slack_nick)
    end
    context.slack_nick = slack_nick
  end
end
```

```ruby
class SendPostEditSlackMessage < ApplicationInteractor
  expects do
    required(:slack_nick).filled
    required(:post).filled
  end

  assures do
  end

  def call
    SlackService.instance.post_message("@#{context.slack_nick} haz updated the post##{context.post.id} with: #{context.post.text}")
  end
end
```

## Les chaînes (organizers)

On se retrouve quand même a transférer un contexte d'un interactor à un autre. Ce n'est pas très pratique.

Pour simplifier la gestion de ces cas, on va utiliser les [organizers](https://github.com/collectiveidea/interactor#organizers).
Les organizers consistent en une succession d'interactors qui transfèrent automatiquement le contexte d'un interactor de la chaine au suivant.

**7: Orchestrez vos commandes avec des organizers pour faire des chaines métier**

On va retravailler cette partie du controller:

```ruby
@post.transaction do
  @post.update!(update_params)

  context = FetchSlackUsername.call(user: current_user)
  SendPostEditSlackMessage.call(slack_nick: context.slack_nick, post: @post)
end
```

On définit un nouvel Interactor pour l'update:

```ruby
class UpdatePost < ApplicationInteractor
  expects do
    required(:post).filled
    required(:update_params).filled
  end

  assures do
  end

  def call
    context.post.update!(context.update_params)
  end
end
```

Puis on défini un organizer pour faire tout ça:

```ruby
class UpdatePostAndPostsToSlack
  # Je met le code ici mais en vrai ça devrait être dans un ApplicationOrganizer ;)
  include Interactor::Organizer
  include Interactor::Contracts
  on_breach do |breaches|
    context.fail!(errors: breaches.flat_map(&:messages), breaches: breaches.flat_map(&:property))
  end

  expects do
    required(:post).filled
    required(:update_params).filled
    required(:user).filled
  end

  assures do
  end

  around do |interactors|
    # Si la moindre exception arrive là dedans, revient en arrière.
    context.post.transaction do
      interactors.call
    end
  end

  organize UpdatePost, FetchSlackUsername, SendPostEditSlackMessage
end
```

Et **enfin** notre controller:

```ruby
class PostsController < ApplicationController
  def update
    # Trouver le post
    @post = Post.find(params[:id])

    # Verifier que le current user à le droit de update
    authorize @post

    # Action !
    UpdatePostAndPostsToSlack.call(user: current_text, update_params: update_params, post: @post)
  end
end
```

## Et les tests dans tout ça ?

Maintenant que tout est découpé en petit bout, tout est beaucoup plus facile à tester. Chaque Interactor, Organizer, Service, Policy pouvant être testé séparément :)

**8: Si c'est pas/difficilement testable ça sent pas bon.**

## Conclusion

Les 8 commandements:

- **1: Dans les contrôleurs, il est possible de définir des méthodes à appeler pour rescue une exception;**
- **2: Pensez à utiliser la version "avec exception" des méthodes à moins que vous n'ayez pas besoin de gérer les échecs;**
- **3: Protégez vous des services externes en les englobant dans des Services;**
- **4: Vos permissions devraient être dans des classes responsables uniquement de leur gestion;**
- **5: Vos règles métier n'ont rien à faire ni dans vos contrôleurs et ni dans vos modèles;**
- **6: Définissez clairement ce qu'attend et ce que va faire chaque interactor;**
- **7: Orchestrez vos commandes avec des organizers pour faire des chaines métier;**
- **8: Si c'est pas/difficilement testable, ça sent pas bon.**

On a vu ce a quoi pouvaient servir les interactors et les différentes techniques pour réduire la taille et DRY ses méthodes de controller.
Toutes ces techniques ne sont pas obligatoires mais ça devrait vous aider quand ça devient trop long ou trop complexe.

