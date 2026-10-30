# TP – Rendre « MiniShop » conforme

Cours 6 – Écrire du code Python sécurisé · durée indicative : 3 h · en binôme ou seul

## Contexte

`app/app.py` est une petite API **FastAPI** volontairement vulnérable. Elle expose des endpoints qui illustrent chacune des failles vues en cours. Votre mission : la rendre conforme, sans casser les fonctionnalités, et mettre en place la CI/CD qui le vérifie en continu.

## Mise en route

```bash
cd "TP06-Secure-Code"
python -m venv .venv && source .venv/bin/activate    # Windows : .venv\Scripts\activate
pip install -r app/requirements.txt pytest bandit pip-audit
cd app && python app.py                               # http://localhost:5000/docs
cd .. && pytest -q tests                              # la plupart des tests sont ROUGES
```

Les tests utilisent `/ping` (Linux/macOS/WSL recommandé). Initialisez un dépôt Git et **faites un commit par faille corrigée** (`git init`, message du type `fix(sqli): requêtes paramétrées`).

## Travail demandé

### Partie 1 – Injections (5 pts)
1. `/login` et `/search` : requêtes paramétrées. ([`3446e1d`](https://github.com/Slyfti/TP06-Secure-Code/commit/3446e1d72e4b4a728cfe213ecc6d10c3d994a8b4))
2. `/ping` : plus de `shell=True`, validation de l'adresse IP (`ipaddress` ou type pydantic `IPvAnyAddress`), timeout. ([`4f0a7ba`](https://github.com/Slyfti/TP06-Secure-Code/commit/4f0a7ba04ac652173b77cc97875cf4d87146a2f6))
3. `/calc` : supprimer `eval` (ex. `ast.literal_eval` ou parser arithmétique à liste blanche), répondre 400/422 si invalide. ([`bed0c18`](https://github.com/Slyfti/TP06-Secure-Code/commit/bed0c1831b159bd2828691e25f4dddc3ba4b1646))
4. `/import-yaml` : `yaml.safe_load`. `/import-profile` : remplacer pickle par du JSON validé par un modèle pydantic. ([`ae82f93`](https://github.com/Slyfti/TP06-Secure-Code/commit/ae82f937b0f49444b63efc166e47a7f09686c467))

### Partie 2 – Fichiers, XSS, SSRF (3 pts)
5. `/files/{name}` : confiner au dossier `uploads/` (`resolve()` + `is_relative_to`). ([`8af95d9`](https://github.com/Slyfti/TP06-Secure-Code/commit/8af95d9cc9a46215743ef1ae87bdd52ae22e9a11))
6. `/hello` : utiliser `Jinja2Templates` (échappement automatique) ou `html.escape`. ([`d59eb56`](https://github.com/Slyfti/TP06-Secure-Code/commit/d59eb56d79b13258134ef4300b69d4e04d88a2d6))
7. `/fetch` : liste blanche d'hôtes, https uniquement, `timeout`, `verify=True`, pas de redirections. ([`c331a66`](https://github.com/Slyfti/TP06-Secure-Code/commit/c331a66205c53690be0967677c4656cbc33df968))

### Partie 3 – Authentification, crypto, secrets (4 pts)
8. Mots de passe hachés avec argon2id (`argon2-cffi`) ou bcrypt, y compris pour les comptes de `init_db()`. ([`76fc875`](https://github.com/Slyfti/TP06-Secure-Code/commit/76fc875a1ba27daa62037ed305e7316205e1c144))
9. `SECRET_KEY` lue depuis la variable d'environnement du même nom (sans valeur par défaut), conservée en constante du module `app.py` ; supprimer la clé AWS en dur ; fournir un `.env.example` et un `.gitignore` qui exclut `.env`. ([`aa1e95b`](https://github.com/Slyfti/TP06-Secure-Code/commit/aa1e95b4559bc24025757d72d33be26229124315))
10. Jeton de session généré avec `secrets` (ou supprimé si inutile) ; cookie de session `Secure`, `HttpOnly`, `SameSite` (`SessionMiddleware(https_only=True, same_site="lax")`). ([`5038c00`](https://github.com/Slyfti/TP06-Secure-Code/commit/5038c00a81d0005ab8638f97dde6ca9e4fcb155b))
11. Ne plus logger les mots de passe. ([`81d9714`](https://github.com/Slyfti/TP06-Secure-Code/commit/81d9714cd185f162e35e95e52870bc034afd5561))

### Partie 4 – Accès, en-têtes, erreurs, configuration (3 pts)
12. `/invoices/{id}` : vérifier que la facture appartient à l'utilisateur connecté (404 sinon), idéalement via une dépendance `Depends(current_user)`. ([`debf2ab`](https://github.com/Slyfti/TP06-Secure-Code/commit/debf2ab7ddd05ac9fd40f902ed935b302ae5d9e5))
12b. **RBAC** : `GET /admin/users` et `DELETE /users/{id}` sont réservés au rôle admin (403 pour un simple utilisateur). Définir les rôles et leurs permissions dans un seul endroit (ex. dépendance `require("user:delete")`), refus par défaut. ([`ee3fdd4`](https://github.com/Slyfti/TP06-Secure-Code/commit/ee3fdd488f11a7ca8519de4b2efc1db5f24dc469))
13. En-têtes de sécurité : CSP, `X-Content-Type-Options: nosniff`, HSTS, `X-Frame-Options` (middleware HTTP `@app.middleware("http")`). ([`69368f3`](https://github.com/Slyfti/TP06-Secure-Code/commit/69368f3087e85cbccda7eeb201186ce9dc6af834))
14. Gestionnaire d'erreurs générique : plus de stack trace ; détail uniquement dans les logs. ([`3a01737`](https://github.com/Slyfti/TP06-Secure-Code/commit/3a017372f28f2f1da1ea82d250c9a6f99e5a5d1c))
15. `FastAPI(debug=False)`, pas de `reload=True`, pas d'écoute sur `0.0.0.0` hors conteneur ; désactiver `/docs` et `/openapi.json` en production. ([`3a01737`](https://github.com/Slyfti/TP06-Secure-Code/commit/3a017372f28f2f1da1ea82d250c9a6f99e5a5d1c), [`f839931`](https://github.com/Slyfti/TP06-Secure-Code/commit/f8399314cce3c66c0ed4357f059dd6e15a893d2d))
16. Bonus : limitation de débit sur `/login` (slowapi), modèles pydantic stricts pour toutes les entrées. ([`57dd030`](https://github.com/Slyfti/TP06-Secure-Code/commit/57dd030665800ef7712c1c6875458c5e23f78ad6))

### Partie 5 – Dépendances et Dockerfile (2 pts)
17. Mettre à jour les dépendances vulnérables (`pip-audit` doit sortir propre) ; produire `requirements.in` + `requirements.txt` verrouillé avec hashes (`pip-compile --generate-hashes`). ([`6c2c02a`](https://github.com/Slyfti/TP06-Secure-Code/commit/6c2c02ad1bd986c6e399f6c6c2bfafbe2d3ad39c), [`bd84629`](https://github.com/Slyfti/TP06-Secure-Code/commit/bd846299b95495c49619162268747e8925c292c8))
18. Dockerfile : image de base épinglée (`python:3.12-slim`), utilisateur non-root (`USER`), `pip install --require-hashes --no-cache-dir`, `.dockerignore`, commande `uvicorn app:app --host 0.0.0.0 --port 5000` sans `--reload`. ([`53812a2`](https://github.com/Slyfti/TP06-Secure-Code/commit/53812a2a0c3dda32f3d62a25e22b69870533790c))

### Partie 6 – CI/CD de sécurité (3 pts)
19. Créer `.github/workflows/security.yml` qui exécute au minimum **Bandit, pip-audit, gitleaks et Trivy**, bloque sur sévérité HIGH/CRITICAL, tourne sur push, pull request **et** en cron quotidien. ([`edc80fd`](https://github.com/Slyfti/TP06-Secure-Code/commit/edc80fd7e33b000a0fd09fa16b9e14369c91fa1a))
20. `permissions: contents: read`, actions épinglées par SHA de commit (les tags peuvent être réécrits : cf. l'incident Trivy/LiteLLM de mars 2026). ([`edc80fd`](https://github.com/Slyfti/TP06-Secure-Code/commit/edc80fd7e33b000a0fd09fa16b9e14369c91fa1a))
21. Activer Dependabot (`.github/dependabot.yml`, écosystèmes `pip` et `docker`). ([`4886bd0`](https://github.com/Slyfti/TP06-Secure-Code/commit/4886bd061c4fad3463547566615da8f3266e2fac))
22. Ajouter un job DAST (OWASP ZAP baseline sur `/openapi.json`) ou un job SBOM (CycloneDX) : au choix. ([`cf1f7f0`](https://github.com/Slyfti/TP06-Secure-Code/commit/cf1f7f078e55e30d2b748c6f6a993a142ef38625))

### Partie 7 – Scénario d'incident (rapport, `INCIDENT.md`)
Une version de `requests` utilisée par l'application est annoncée comme compromise. Décrivez, avec les commandes, votre réaction : détection, évaluation d'exposition, confinement, correction, rotation des secrets, retour d'expérience. Ce rapport est évalué à l'oral.

## Vérification

```bash
bash verify.sh        # Bandit, pip-audit, gitleaks, pytest
```

## Barème (20 points)

| Bloc | Points | Contrôle |
|---|---|---|
| Injections | 5 | tests `sqli`, `command`, `eval`, `yaml`, `pickle` |
| Fichiers, XSS, SSRF | 3 | tests `path_traversal`, `xss`, `ssrf` |
| Auth, crypto, secrets | 4 | tests mots de passe, secret key, AWS, cookies, logs ; gitleaks propre |
| Accès, RBAC, headers, erreurs, config | 3 | tests `idor`, `rbac`, headers, stacktrace, debug |
| Dépendances et Dockerfile | 2 | pip-audit propre, hashes, Dockerfile durci |
| CI/CD de sécurité | 3 | workflow présent et complet, Dependabot |

Les tests fonctionnels (`test_functional_*`) doivent rester verts : une correction qui casse l'application ne rapporte aucun point.

## Pistes et rappels

- Argon2 : `from argon2 import PasswordHasher`, `ph.hash(...)`, `ph.verify(hash, pwd)`.
- Le client de test utilise `https://testserver` pour que le cookie `Secure` soit renvoyé.
- Le test des cookies lit l'en-tête `Set-Cookie` de `/login` : il doit contenir `HttpOnly`, `Secure` et `SameSite`.
- Le test des hashes exige `--hash=sha256:` dans `app/requirements.txt`.
- Les codes 422 de validation pydantic sont acceptés pour les entrées invalides.
