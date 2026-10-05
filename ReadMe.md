# recherche_immobilier.py

Script autonome de veille immobilière (achat et location) à **L'Aigle (61300)**, exécuté sur un Raspberry Pi 5, à partir du site "Se Loger".

## Principe
Aucun accès au site SeLoger : le script ne scrape rien (interdit par les conditions d'utilisation de SeLoger). Il lit uniquement **votre boîte Gmail** (IMAP), où SeLoger vous envoie ses alertes d'annonces pour vos recherches enregistrées — un service gratuit explicitement prévu par SeLoger. Les liens de suivi contenus dans les e-mails ne sont **jamais suivis** par le script : ils sont seulement conservés, un clic de votre part ouvre l'annonce.

Exécution **unique** à chaque lancement (pas de boucle interne) : la récurrence est assurée par cron, pas par le script.

## Coexistence avec emails_scan.py
Un filtre Gmail dédié (`De : seloger` → libellé **Immo**, « Ignorer la boîte de réception ») détourne les alertes SeLoger avant qu'elles n'atteignent la boîte de réception. `emails_scan.py` ne traite que la boîte de réception (messages non lus) et transfère/vide tout vers Outlook : avec ce filtre, il ne voit jamais les alertes SeLoger. `recherche_immobilier.py` les lit séparément, dans le libellé Immo. **Aucune modification d'`emails_scan.py` n'est requise.**

Les deux scripts réutilisent volontairement les mêmes conventions : même mot de passe d'application Gmail (`GMAIL_APP_PW`, dans le même `.secrets.env`), et même bot/fichier Telegram (`~/.telegram_config`).

## Fonctionnement
1. Lecture (en lecture seule, `BODY.PEEK`, jamais marqué comme lu) de tous les messages du libellé Immo dans la fenêtre `jours_max`.
2. Extraction des annonces de chaque e-mail (prix, type de bien, pièces, surface, quartier/ville, code postal, lien) à partir du HTML de l'e-mail.
3. Déduction de la transaction (achat/location) à partir du texte de l'annonce (mention louer/vendre, suffixe `/mois`, prix au m²), du sujet de l'e-mail, ou d'un bandeau « Votre recherche » présent ailleurs dans le message.
4. Filtrage selon vos critères (`recherche_immobilier.yaml`), par transaction.
5. Comparaison avec la dernière version connue de chaque annonce (via une empreinte, voir plus bas) : seules les annonces **nouvelles** ou dont le **prix a changé** sont journalisées.
6. Écriture d'un CSV (historique, ajout seul) et d'une page HTML (toutes les annonces connues, réécrite à chaque exécution), puis notification Telegram si pertinent.

## Mise en place (une seule fois)
1. **SeLoger** : compte gratuit, créer une ou plusieurs recherches enregistrées avec alerte e-mail (achat et/ou location) pour L'Aigle et vos critères.
2. **Gmail** : Paramètres > Filtres et adresses bloquées > Créer un filtre
   ```
   De : seloger          (laisser les autres champs vides)
   ```
   puis « Créer un filtre » et cocher :
   - « Ignorer la boîte de réception (l'archiver) »
   - « Appliquer le libellé » > Nouveau libellé : **Immo**
3. Le script réutilise le mot de passe d'application Gmail déjà stocké pour `emails_scan.py` (variable `GMAIL_APP_PW`, lue dans l'environnement ou dans le fichier `.secrets.env` indiqué dans la configuration).
4. Premier lancement : le script crée `recherche_immobilier.yaml` (modèle), puis s'arrête. Éditez-le (critères, codes postaux), puis relancez.
5. **Telegram (facultatif)** : réutilise le même bot et le même fichier que `emails_scan.py` (`~/.telegram_config`, section `[telegram]`, clés `token_groq`/`chat_id`) — rien à reconfigurer si `emails_scan.py` l'utilise déjà.

## Outils de mise au point
```bash
python3 recherche_immobilier.py --eml alerte.eml    # teste l'extraction sur un e-mail enregistré (.eml/.html), sans Gmail ni écriture
python3 recherche_immobilier.py --a-blanc           # lit Gmail, affiche le résultat, n'écrit rien
python3 recherche_immobilier.py --diagnostic        # affiche, pour CHAQUE e-mail lu, expéditeur/sujet/contenu brut ; montre aussi les e-mails écartés
python3 recherche_immobilier.py --tout              # ignore les critères de recherche_immobilier.yaml
python3 recherche_immobilier.py --depuis-jours N    # force la fenêtre de lecture (par défaut : jours_max)
```

## À savoir
- Le statut lu/non lu des e-mails n'a aucune importance : le script lit tous les messages du libellé Immo dans la fenêtre `jours_max`, qu'ils soient lus ou non, et ne les marque jamais comme lus.
- Si des alertes reçues dans le libellé ne ressortent pas, lancez `--diagnostic` : il montre chaque message lu (annonce trouvée ou non) et ceux écartés faute d'expéditeur reconnu. Un message compté « analysé » mais à 0 annonce n'est pas forcément une anomalie — SeLoger envoie aussi des e-mails de promotion de son application, sans annonce exploitable.
- Lancez le script deux fois de suite (`--a-blanc`) : la 2e fois doit annoncer 0 nouveauté.
- Testez avec un environnement proche de celui de cron (qui ne reprend pas votre session interactive) :
  ```bash
  env -i HOME="$HOME" python3 recherche_immobilier.py --a-blanc
  ```
  S'il échoue à trouver `.secrets.env`, ajoutez une ligne `HOME=/home/xxx` en tête de votre crontab, avant la ligne de commande.
- Ajoutez `>> journal.log 2>&1` à la fin de la ligne de cron pour garder une trace des exécutions et diagnostiquer un échec silencieux.

## Limites à connaître
- Une alerte ne signale que les **nouvelles** annonces : le script ne peut pas savoir qu'une annonce a disparu (vendue/louée).
- Un changement de prix n'est détecté que si SeLoger renvoie l'annonce dans une alerte ultérieure.
- L'identité d'une annonce est une **empreinte** (transaction, type, pièces, surface, code postal, quartier, ville) : le lien de suivi est propre à chaque e-mail et ne peut pas servir d'identifiant. Deux biens rigoureusement identiques dans le même quartier seraient confondus.
- Un e-mail de confirmation (envoyé à la création d'une recherche) est toujours compté à part (« confirmation(s) ignorée(s) »), mais ses éventuelles « annonces similaires » sont lues comme celles de n'importe quel e-mail, selon le réglage `inclure_similaires` — c'est le moyen de récupérer un premier aperçu après avoir créé ou recréé une recherche.

## Configuration (`recherche_immobilier.yaml`)
```yaml
gmail:
  host: imap.gmail.com
  port: 993
  user: jfbconseil14@gmail.com
  dossier: Immo                         # libellé Gmail créé par le filtre
  mot_de_passe_env: GMAIL_APP_PW        # jamais la valeur en clair ici
  fichier_secrets: ~/Projects/Groq_agent/Scan_emails/.secrets.env

expediteurs:
  - seloger

jours_max: 90                  # fenêtre de lecture des e-mails (jours)
inclure_similaires: false      # true : récupère aussi les « annonces similaires »
                               # (hors critères stricts) glissées dans un e-mail
                               # de confirmation — utile après une recréation
                               # de recherche, à remettre à false ensuite

criteres:
  location:
    pieces_max: 3
    budget_max: 600
    codes_postaux: ["61300"]
  achat:
    pieces_max: 3
    budget_max: 200000
    codes_postaux: ["61300"]

telegram:
  config_path: ~/.telegram_config   # même fichier/bot que emails_scan.py
  notifier_si_rien: false           # true : notifie aussi quand il n'y a rien de nouveau
```
Tous les critères sont facultatifs ; une valeur absente sur une annonce (ex. pièces non indiquées) n'exclut pas l'annonce.

## Installation
```bash
pip install pyyaml --break-system-packages   # déjà installé pour emails_scan.py
```

## Arborescence du projet
```
recherche_immobilier.py               script principal
recherche_immobilier.yaml             configuration et critères
recherche_immobilier_historique.csv   journal des nouveautés (ajout seul) ;
                                       colonne 'ouvrir' = formule tableur
                                       (=LIEN.HYPERTEXTE(...), nom français de HYPERLINK)
recherche_immobilier_annonces.html    TOUTES les annonces connues, avec de vrais liens
                                       cliquables (<a>) ; réécrit à chaque exécution
                                       (pas un journal) — à privilégier pour la consultation,
                                       sans dépendre des réglages d'import CSV d'un tableur
recherche_immobilier_etat.json        e-mails déjà traités (à vider pour relancer le processus)
journal.log                           sortie cron (si configuré en >> journal.log 2>&1)
```

## Notification Telegram
Après chaque exécution réelle (pas `--a-blanc`), un message est envoyé dès qu'au moins une nouveauté ou un changement de prix a été trouvé — type, prix, pièces, surface, lieu et lien pour chaque nouveauté. Passez `telegram.notifier_si_rien` à `true` pour être notifié même quand il n'y a rien de nouveau. Si le fichier ou la section `[telegram]` est absent, la notification est simplement ignorée (avertissement sur stderr) : ce n'est jamais bloquant.

## Projets associés
- **`emails_scan.py`** — scan/classification des 3 boîtes email JFBConseils. Projet distinct, avec lequel celui-ci partage le mot de passe d'application Gmail et le bot Telegram (voir « Coexistence » plus haut).

## Auteur
Jean-François Brunet – JFBConseils - Octobre 2026
