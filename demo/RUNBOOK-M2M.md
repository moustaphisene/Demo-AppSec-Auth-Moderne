# Runbook — Authentification machine-à-machine (`client_credentials`)

> Support de démonstration : comment deux microservices s'authentifient **sans aucun utilisateur
> dans la boucle**, et pourquoi c'est un modèle d'identité différent — pas une variante du login.
>
> Pour la liste des fichiers changés et la procédure de déploiement courte, voir
> [`keycloak/README.md`](keycloak/README.md).

---

## 1. Ce que vous démontrez

Deux appels vers le **même** service aval (`:8091`), avec deux identités radicalement différentes.

| | Identité **déléguée** | Identité **machine** |
|---|---|---|
| Grant OAuth2 | `authorization_code` | `client_credentials` |
| Qui est le sujet (`sub`) ? | l'utilisateur `testuser` | le *service account* du client |
| Navigateur / redirect_uri | requis | **aucun** |
| Secret client | requis (client confidentiel) | requis (il n'y a rien d'autre pour s'authentifier) |
| `refresh_token` | oui | **non** — la machine redemande un jeton |
| Porteur du rôle | l'utilisateur (rôle *realm*) | le client (rôle *client*) |
| Endpoint autorisé | `/planets` | `/internal/planets` |
| Révocation | déconnexion / session | désactiver le client ou tourner le secret |

Le point pédagogique central : **authentifier n'est pas autoriser**. Activer les *service accounts*
donne un jeton ; sans rôle affecté au service account, ce jeton valide prend un **403**.

---

## 2. Pourquoi `client_credentials` ne marchait pas dans ce POC

Quatre couches défaillantes, dans cet ordre. Messages d'erreur réellement observés sur ce realm :

| # | Couche | Symptôme | Cause |
|---|---|---|---|
| 1 | **Keycloak** | `401 {"error":"unauthorized_client","error_description":"Public client not allowed to retrieve service account"}` | `keycloak-angular` est public : pas de secret, donc pas d'identité machine possible |
| 2 | **Keycloak** | `401 {"error":"unauthorized_client","error_description":"Invalid client or Invalid client credentials"}` | aucun client du realm n'avait `serviceAccountsEnabled` — il n'existait **aucune** identité machine |
| 3 | **Image Docker** | `COPY keycloak/import/realm-export.json` échoue au build | l'export était à `demo/realm-export.json`, le `Dockerfile` l'attendait à `demo/keycloak/import/` |
| 4 | **Modèle d'autorisation** | `403` sur `/planets` même avec un jeton valide | `/planets` exige le rôle *realm* `user` : un service account ne l'a pas, et ne doit pas l'avoir |

> Corollaire : cocher « Service accounts roles » dans la console n'aurait rien résolu. Le jeton
> aurait été délivré, puis refusé en aval. La vraie correction est de **modéliser une identité
> machine** : un client dédié, un rôle dédié, une audience, un endpoint dédié.

---

## 3. Ce qui a été mis en place

```
Keycloak (realm bzhcamp)
├── app-m2m             ← client confidentiel, Service accounts ACTIVÉ
│   ├── pas de standardFlow, pas de directAccessGrants  (aucun chemin humain)
│   ├── mapper d'audience → "service-planets"
│   └── service-account-app-m2m  →  rôle client planets.read
├── service-planets     ← représente l'API aval : porte les rôles machine, sert d'audience
├── app-springboot      ← inchangé (flux navigateur) + mapper d'audience
└── keycloak-angular    ← inchangé (public) + mapper d'audience

service-springboot-rest (:8091)
├── GET /planets            → hasAuthority("ROLE_realm_user")                         [humain]
└── GET /internal/planets   → hasAuthority("ROLE_service-planets_planets.read")       [machine]

app-springboot (:8090)
├── registration "keycloak"      → authorization_code   → WebClient + filtre OAuth2 (session utilisateur)
└── registration "keycloak-m2m"  → client_credentials   → jeton demandé explicitement, hors session
    └── GET /m2m/planets  → appelle /internal/planets avec un jeton machine
```

Fichiers concernés :

| Fichier | Rôle |
|---|---|
| `demo/keycloak/import/realm-export.json` | realm déclaratif (source unique de vérité) |
| `demo/keycloak/apply-m2m.sh` | applique les mêmes objets à un realm **déjà démarré** (non destructif) |
| `service-springboot-rest/.../web/InternalPlanetsController.java` | API machine |
| `service-springboot-rest/.../config/SecurityConfig.java` | règle d'autorisation `/internal/**` |
| `app-springboot/.../config/M2mClientConfig.java` | manager `client_credentials` hors session + les pièges documentés |
| `app-springboot/.../service/M2mPlanetsService.java` | appel sortant machine (jeton demandé explicitement) |
| `app-springboot/src/main/resources/application.yml` | registration `keycloak-m2m` |
| `postman/` dossier **09** | matrice d'accès testée (4 cases) |

---

## 4. Mise en place

### Chemin A — realm neuf (recommandé pour préparer la démo)

`--import-realm` n'importe **que** sur une base vide. Si `demo_pgdata` existe, l'import est ignoré.

```bash
cd demo
docker compose down -v          # DÉTRUIT la base Keycloak (realm, utilisateurs, secrets)
docker compose up -d --build    # rebuild + import du realm avec l'identité machine
```

### Chemin B — realm existant (à privilégier en salle : non destructif)

```bash
cd demo
docker compose up -d
./keycloak/apply-m2m.sh         # idempotent : relançable sans effet de bord
```

Le script affiche en sortie le secret du client `app-m2m`.

### Puis, dans les deux cas

```bash
export M2M_CLIENT_SECRET='<secret affiché par apply-m2m.sh>'
export KEYCLOAK_CLIENT_SECRET='<secret du client app-springboot>'

cd demo/service-springboot-rest && mvn spring-boot:run   # :8091
cd demo/app-springboot         && mvn spring-boot:run    # :8090
```

---

## 5. La démonstration, en direct

### 5.1 Obtenir un jeton machine — l'étape qui ne marchait pas

```bash
M2M_TOKEN=$(curl -s -X POST \
  http://localhost:8080/realms/bzhcamp/protocol/openid-connect/token \
  -d grant_type=client_credentials \
  -d client_id=app-m2m \
  -d client_secret="$M2M_CLIENT_SECRET" | jq -r .access_token)

echo "${M2M_TOKEN:0:40}..."
```

Montrer le contenu — c'est là que le modèle devient visible :

```bash
echo "$M2M_TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq '{
  sub, azp, aud, preferred_username,
  realm_roles: .realm_access.roles,
  client_roles: .resource_access
}'
```

À commenter à voix haute :
- `preferred_username` = `service-account-app-m2m` → **le sujet n'est pas un humain**
- `resource_access["service-planets"].roles` = `["planets.read"]` → **l'autorisation est portée par le client**
- `aud` contient `service-planets` → **le jeton nomme son destinataire**
- aucun `refresh_token` dans la réponse → **une machine se réauthentifie, elle ne rafraîchit pas**

### 5.2 Le jeton machine ouvre l'API machine

```bash
curl -s -H "Authorization: Bearer $M2M_TOKEN" \
  http://localhost:8091/internal/planets | jq
```
→ `200`, et la réponse rappelle quelle identité a été reconnue (`caller.authorizedParty`).

### 5.3 Le même jeton est refusé sur l'API humaine — **le moment clé**

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $M2M_TOKEN" http://localhost:8091/planets
```
→ `403`. Le jeton est valide : signature, issuer, expiration, tout est bon. C'est l'**autorisation**
qui refuse. Insister : ce 403 n'est pas un bug à corriger, c'est la démonstration que les deux
identités sont étanches.

### 5.4 Et l'inverse (sinon on n'a prouvé que la moitié)

```bash
HUMAN_TOKEN=$(curl -s -X POST \
  http://localhost:8080/realms/bzhcamp/protocol/openid-connect/token \
  -d grant_type=password -d client_id=app-springboot \
  -d client_secret="$KEYCLOAK_CLIENT_SECRET" \
  -d username=testuser -d password=password -d scope=openid | jq -r .access_token)

curl -s -o /dev/null -w 'humain  → /planets          : %{http_code}\n' \
  -H "Authorization: Bearer $HUMAN_TOKEN" http://localhost:8091/planets
curl -s -o /dev/null -w 'humain  → /internal/planets : %{http_code}\n' \
  -H "Authorization: Bearer $HUMAN_TOKEN" http://localhost:8091/internal/planets
curl -s -o /dev/null -w 'machine → /planets          : %{http_code}\n' \
  -H "Authorization: Bearer $M2M_TOKEN" http://localhost:8091/planets
curl -s -o /dev/null -w 'machine → /internal/planets : %{http_code}\n' \
  -H "Authorization: Bearer $M2M_TOKEN" http://localhost:8091/internal/planets
curl -s -o /dev/null -w 'anonyme → /internal/planets : %{http_code}\n' \
  http://localhost:8091/internal/planets
```

### 5.5 La matrice d'accès (le tableau à laisser à l'écran)

| appelant | `/planets` (humain) | `/internal/planets` (machine) |
|---|---|---|
| jeton humain `testuser` | **200** | **403** |
| jeton machine `app-m2m` | **403** | **200** |
| sans jeton | **401** | **401** |

### 5.6 Côté application : le même code, un `WebClient` différent

```bash
curl -s http://localhost:8090/m2m/planets    # nécessite d'être authentifié sur :8090
```

Montrer côte à côte `PlanetsService` (jeton de l'utilisateur, lu en session) et `M2mPlanetsService`
(jeton du client, obtenu et mis en cache hors session). Le code métier est identique ; seule
l'injection change. C'est l'argument « la façon d'obtenir le jeton est une décision de
configuration, pas de code métier ».

### 5.7 Suite automatisée (Postman / Newman)

```bash
cd postman
export CLIENT_SECRET="$KEYCLOAK_CLIENT_SECRET"
export M2M_CLIENT_SECRET="$M2M_CLIENT_SECRET"
./run-newman.sh        # dossier 09 = les 4 cases de la matrice
```

---

## 6. Dépannage — erreur observée → cause réelle

| Réponse | Cause | Correction |
|---|---|---|
| `401 unauthorized_client` / *Public client not allowed to retrieve service account* | le client est public | utiliser un client **confidentiel** ; un client public ne peut pas détenir de secret |
| `401 unauthorized_client` / *Client not enabled to retrieve service account* | `serviceAccountsEnabled = false` | activer « Service accounts roles » (ou lancer `apply-m2m.sh`) |
| `401 invalid_client` / *Invalid client or Invalid client credentials* | client inexistant **ou** secret erroné — Keycloak ne distingue pas volontairement | vérifier `client_id`, puis relire le secret via `apply-m2m.sh` |
| `400 unsupported_grant_type` | grant non autorisé sur le client | vérifier le client ; en Spring, vérifier `authorization-grant-type: client_credentials` |
| Jeton obtenu mais **403** en aval | le service account n'a pas le rôle attendu | affecter `planets.read` (client `service-planets`) à `service-account-app-m2m` |
| **403** avec le rôle affecté | `resource_access` absent du jeton | `fullScopeAllowed` désactivé, ou rôle affecté au mauvais sujet (client ≠ service account) |
| **401** `invalid_token` en aval | issuer/audience | le service valide `http://localhost:8080/realms/bzhcamp` ; si `KEYCLOAK_AUDIENCE` est renseigné, le mapper d'audience doit exister |
| Modification du realm **sans effet** | `--import-realm` ne rejoue pas sur une base existante | `docker compose down -v` (destructif) ou `apply-m2m.sh` (non destructif) |
| `mvn spring-boot:run` : `UnsupportedClassVersionError ... compiled by a more recent version ... recognizes class file versions up to 55.0` | `mvn` tourne sous JDK 11 (classfile 55) ; ces modules exigent JDK 17 (classfile 61, Spring Boot 3.3) | `export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home` avant `mvn spring-boot:run`, dans **chaque** terminal |
| Spring : `IllegalArgumentException: clientSecret cannot be empty` | `M2M_CLIENT_SECRET` non exporté | exporter la variable avant `mvn spring-boot:run` |
| Spring : `IllegalStateException: Impossible d'obtenir un jeton client_credentials` | secret erroné ou service accounts désactivé côté Keycloak | message volontairement explicite — voir `M2mPlanetsService.machineAccessToken()` ; vérifier `M2M_CLIENT_SECRET` et `apply-m2m.sh` |

---

## 7. Pièges à connaître avant de monter sur scène

| Piège | Symptôme en salle | À faire |
|---|---|---|
| `clockSkew` de 60 s du provider `client_credentials` | vous réduisez la durée de vie des jetons à 30 s pour montrer l'expiration → un appel au token endpoint **à chaque requête**, et la salle conclut « le cache ne marche pas » | garder une durée de vie > 60 s, ou démontrer le phénomène exprès : c'est une bonne leçon |
| Filtre `WebClient` OAuth2 pour du M2M | le jeton est mis en cache **par appelant entrant** (un aller-retour Keycloak par appelant), et la chaîne casse dans un `@Scheduled` faute d'`Authentication` | demander le jeton explicitement avec un principal constant — c'est ce que fait `M2mPlanetsService`, et le javadoc explique pourquoi |
| `registrationId` par défaut oublié sur un filtre OAuth2 | le filtre n'attache **rien, sans erreur** → 401 en aval, et on cherche une heure du côté de Keycloak | asserter la présence de l'en-tête `Authorization` dans un test |
| Bloc `caller` de `/internal/planets` | révèle la taxonomie d'autorisation (`ROLE_<client>_<role>`) ; capté par les logs de corps de réponse et les captures d'écran | annoncer « pédagogique, pas production » ; en vrai, ne jamais énumérer les autorités dans une réponse |
| `/m2m/planets` sur `:8090` | c'est l'appel **sortant** qui est machine ; l'appel entrant reste soumis à l'authentification de l'app — sinon on publie un proxy ouvert vers l'API interne | le dire explicitement, c'est une confusion fréquente |

---

## 8. Aller plus loin (si la salle suit)

- **Activer la validation d'audience.** Les mappers sont en place ; il suffit de démarrer le service
  avec `KEYCLOAK_AUDIENCE=service-planets`. Puis retirer le mapper d'un client et montrer le `401` :
  c'est la défense contre le *confused deputy* (un jeton émis pour l'API A rejoué contre l'API B).
- **Durée de vie.** Réduire l'`accessTokenLifespan` du client machine : sans refresh token, le coût
  d'un jeton court est seulement un appel de plus au token endpoint — tolérance bien meilleure
  qu'avec une session humaine.
- **Rotation du secret.** Régénérer le secret dans la console et montrer l'échec immédiat, puis le
  rétablissement par simple variable d'environnement : la rotation n'exige aucun déploiement de code.
- **Ce que `client_credentials` NE fait pas.** Il n'y a aucune identité utilisateur dans le jeton :
  impossible de tracer « quel utilisateur » a déclenché l'action en aval. Si c'est nécessaire,
  le sujet est le *token exchange* (RFC 8693) / on-behalf-of — un autre modèle, une autre leçon.
- **Un client par appelant.** Ne pas cumuler flux navigateur et identité machine dans un même
  client : `azp` devient inexploitable en audit et un secret fuité compromet les deux surfaces.

---

## 9. Détail observé : les deux chemins ne produisent pas un jeton strictement identique

Mesuré sur les deux chemins de mise en place, avec le même client `app-m2m` :

| | Chemin A (import déclaratif) | Chemin B (`apply-m2m.sh` / kcadm) |
|---|---|---|
| `resource_access` | `{"service-planets":{"roles":["planets.read"]}}` | idem **+** `account` |
| `realm_access` | *absent* | `offline_access, uma_authorization, default-roles-bzhcamp` |
| `aud` | `service-planets` | `service-planets, account` |

Pourquoi : un service account créé par l'Admin API reçoit le rôle composite
`default-roles-<realm>` du realm, alors que l'import déclaratif n'applique que ce que le fichier
déclare — et le fichier ne déclare aucun rôle realm pour `service-account-app-m2m`.

Conséquence pratique : **aucune** sur la démonstration (ni l'un ni l'autre ne porte le rôle `user`,
les deux prennent donc 403 sur `/planets`). Conséquence pédagogique : le chemin déclaratif est
*plus strict par défaut* — un bon argument pour versionner le realm plutôt que de cliquer dans la
console. Pour retirer ces rôles hérités côté kcadm :

```bash
docker exec -i demo-keycloak-1 /opt/keycloak/bin/kcadm.sh remove-roles -r bzhcamp \
  --uusername service-account-app-m2m --rolename default-roles-bzhcamp \
  --config /tmp/kcadm-m2m.json
```
