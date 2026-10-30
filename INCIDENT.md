# Rapport d'incident – `requests` compromis

> Scénario fictif : la version `requests==2.34.2` publiée sur PyPI est annoncée comme compromise

## 1. Détection

Alerte reçue via Dependabot / GitHub Advisory, ou par le job `pip-audit` du workflow quotidien.

```bash
pip-audit -r app/requirements.txt --strict --desc   # confirme la vulnérabilité, avec sa description
grep -A1 '^requests==' app/requirements.txt         # version et hash épinglés
```

## 2. Évaluation de l'exposition

- Le module `requests` est utilisé dans `app/app.py` pour l'endpoint `/fetch`. Il est importé au démarrage, donc le code malveillant s'exécute même si `/fetch` n'est jamais appelé.
- Les SBOM indiquent quels builds sont concernés par la version compromise:

```bash
git log --oneline -S 'requests==2.34.2' -- app/requirements.txt   # depuis quand ?
gh run list --workflow security.yml                               # runs/builds concernés
gh run download <run-id> -n sbom && grep -A2 '"name": "requests"' sbom.cdx.json
docker run --rm minishop:prod pip show requests                   # image en production
pip freeze | grep -i '^requests=='                                # postes de dev et venv locaux
```

- Traiter comme suspect tout build ou `pip install` fait entre la publication de la version et son retrait de PyPI.

Rechercher les traces (indicateurs donnés dans l'avis) :

```bash
find / -path "*site-packages/*.pth" 2>/dev/null                  # .pth exécutés au démarrage de Python : vérifier les inconnus
ls ~/.config/systemd/user/                                       # persistance via un service utilisateur
grep -r "<domaine-exfiltration>" /var/log 2>/dev/null            # connexions vers le domaine de l'attaquant
```

- Tout ce qui est dans l'environnement du conteneur ou de la CI a pu fuiter : `SECRET_KEY`, et en CI le `GITHUB_TOKEN` (en lecture seule grâce à `permissions: contents: read`).
- Les hashes (`--require-hashes`) protègent contre un paquet modifié après le verrouillage, mais pas contre une version malveillante publiée puis verrouillée par nous.

## 3. Confinement

```bash
docker stop minishop && docker rm minishop        # arrêter les conteneurs touchés
```

- Geler les déploiements et les merges (Dependabot compris) jusqu'à la correction.
- Bloquer la version compromise pour qu'elle ne soit plus réinstallée nulle part (par exemple `requests!=2.34.2` dans `requirements.in`).
- Isoler les hôtes touchés et supprimer la persistance éventuelle : un simple upgrade ne retire pas un malware déjà installé.
- Bloquer les sorties réseau du conteneur (seul `example.com` est légitime pour `/fetch`) et rechercher dans les logs du proxy/pare-feu des connexions sortantes inhabituelles.

## 4. Correction

Épingler une version saine (antérieure ou corrigée), reverrouiller avec les hashes, reconstruire et revérifier :

```bash
cd app
pip-compile --generate-hashes --upgrade-package 'requests==2.34.1' -o requirements.txt requirements.in
cd .. && pip-audit -r app/requirements.txt --strict && pytest -q tests
docker build --pull -t minishop:fixed app
docker run --rm -v "$PWD:/src" aquasec/trivy fs --severity HIGH,CRITICAL /src
git commit -am "fix(deps): pin requests to safe version (incident)"
```

Redéployer uniquement après un workflow CI entièrement vert.

- Inspecter la version suspecte sans l'exécuter : `pip download requests==2.34.2 --no-deps -d /tmp/suspect`, puis lire le contenu du wheel.
- Purger les caches pour ne pas réinstaller le paquet malveillant : `pip cache purge`, et reconstruire les images avec `--no-cache` sur un environnement neuf, à partir d'un état connu sain.
- Si pas de version saine disponible : désactiver la fonctionnalité (`/fetch`) ou retirer la bibliothèque.

## 5. Rotation des secrets

Considérer comme compromis tout secret accessible au build et à l'exécution (variables d'environnement, clés SSH, identifiants cloud, tokens CI), y compris sur les postes de dev :

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"   # nouvelle SECRET_KEY
```

- Remplacer `SECRET_KEY` en production : toutes les sessions existantes deviennent invalides (déconnexion forcée, voulue).
- Révoquer et recréer tout autre identifiant présent dans l'environnement (clés cloud, tokens d'API, secrets GitHub Actions).
- Par précaution, forcer la réinitialisation des mots de passe si une fuite de la base est possible (les hashes argon2id limitent l'impact).

## 6. Retour d'expérience

- Ce qui a fonctionné : le cron quotidien (pip-audit + Trivy) et Dependabot ont détecté l'alerte, le SBOM a permis de savoir rapidement quelles builds étaient touchées, et le `GITHUB_TOKEN` en lecture seule a limité l'impact en CI.
- Points à améliorer:
  - Retarder l'adoption des nouvelles versions (ex. `cooldown` Dependabot de quelques jours).
  - Filtrer les sorties réseau du conteneur en production.
  - Stocker les secrets dans un coffre (Vault, secrets manager) avec rotation automatique.
  - Ajouter une règle de CI tirée de l'incident et fixer un SLA de correction (délai maximal pour corriger une vulnérabilité selon sa sévérité).
