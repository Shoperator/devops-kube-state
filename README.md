# devops-kube-state

Deklarativno stanje klastera (GitOps preko ArgoCD). App-of-apps root:
`clusters/local/root.yaml`. Child aplikacije su u `clusters/local/apps/`.

## Redosled (sync-wave)

| wave | aplikacija | zašto tu |
| --- | --- | --- |
| 0 | `ingress-nginx` | Objavljuje ShopHub i sve prodavnice |
| 0 | `kube-prometheus-stack` | Prometheus, Grafana i njihovi CRD-ovi |
| 0 | `cnpg` | Operator za PostgreSQL -- `standard` izbor baze |
| 0 | `redis-operator` | Operator za Redis -- `light` izbor baze |
| 1 | `shop-operator` | Instalira Shop/DiscordChannel/Wallet CRD-ove i pokreće operator |
| 1 | `observability-dashboards` | Grafana dashboard po prodavnici |
| 2 | `shophub` | Kreira Shop resurse, pa CRD-ovi iz wave 1 moraju već postojati |

## Domen prodavnica

`shop-operator` i `shophub` moraju dobiti **istu** vrednost `shopBaseDomain`:

- operator: prosleđuje je kao `--shop-base-domain` i upisuje Ingress host
- shophub: dobija je kao `SHOP_BASE_DOMAIN` i pravi link u dashboard-u

Ako se razlikuju, prodavnica radi ali link iz dashboard-a vodi na host koji
Ingress ne objavljuje. Trenutna vrednost je `localhost`: browser sam razrešava
svako ime pod `localhost` na loopback adresu, pa je prodavnica dostupna čim je
deploy-ovana -- bez DNS zapisa i bez ručnog upisa u hosts fajl. Radi i bilo koji
wildcard DNS domen (`127.0.0.1.sslip.io`), ili pravi domen.

Da bi to stiglo do browsera, `ingress-nginx` je podešen sa `hostPort` -- kind
objavljuje portove 80 i 443 control-plane čvora na host, pa nešto mora da sluša
na portu 80 samog čvora. NodePort Service dobija port iz opsega 30000-32767,
koji niko ne prosleđuje.

## Baze prodavnica

Prodavnica bira bazu pri kreiranju, a `shop-operator` je ne deploy-uje sam --
upisuje custom resource koji operator te baze prati:

| izbor u ShopHub-u | `spec.database` | resurs | operator |
| --- | --- | --- | --- |
| standard | `postgresql` | `Cluster` (postgresql.cnpg.io) | `cnpg` |
| light | `redis` | `Redis` (redis.redis.opstreelabs.in) | `redis-operator` |

Ako operator baze nije instaliran, prodavnica se svejedno deploy-uje i čeka
svoju bazu.

## Bootstrap klastera

Klaster se pravi iz `clusters/local/kind-config.yaml`, koji je deo ovog
repozitorijuma jer je **preduslov** za `apps/ingress-nginx.yaml`. Kontroler se
vezuje za `hostPort` 80/443 i bira cvor preko `nodeSelector: ingress-ready`, a
oba se podesavaju samo pri kreiranju klastera -- na postojecem klasteru se ne
mogu dodati. Klaster napravljen bez ovog fajla ostavlja ingress kontroler u
Pending stanju.

```bash
kind create cluster --config clusters/local/kind-config.yaml

kubectl create namespace argocd
# --server-side je obavezan: applicationsets CRD je veci od 262144 bajta,
# koliko staje u anotaciju koju klijentski `apply` upisuje uz resurs. Bez
# njega prolazi sve osim tog jednog CRD-a, i to bez prekida instalacije.
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server --timeout=5m

# registruje OCI registry sa kojeg se povlace chart-ovi
kubectl apply -f clusters/local/argocd/repo-ghcr-charts.yaml

# vidi "Pre prve sinhronizacije" ispod
kubectl create namespace shophub
kubectl create secret generic shophub-auth -n shophub --from-literal=JWT_SECRET="$(openssl rand -hex 32)"

# app-of-apps -- sve ostalo ide odavde
kubectl apply -f clusters/local/root.yaml
kubectl -n argocd get applications -w
```

Provera da je wave 0 zaista prosao:

```bash
kubectl -n ingress-nginx get pods -o wide   # Running, NODE = shophub-control-plane
curl -i http://localhost/                   # 404 od nginx-a znaci da slusa
```

## Pre prve sinhronizacije

ShopHub potpisuje access token-e ključem iz Secret-a. ArgoCD renderuje chart bez
pristupa klasteru, pa bi generisan ključ bio nov pri svakoj sinhronizaciji i
odjavio bi sve korisnike. Zato se pravi jednom, ručno, i to **pre** nego što
`root.yaml` pusti `shophub` aplikaciju — komande su u bootstrap bloku iznad.

Ključ nije u ovom repozitorijumu i ne vraća ga nijedan sync: novi klaster znači
novi ključ, i time odjavu svih postojećih korisnika.

## Struktura

```
kube-state
└── clusters/
    └── local/
        ├── cluster.yaml          # metapodaci klastera
        ├── kind-config.yaml      # ulaz za `kind create cluster` (nije ArgoCD)
        ├── root.yaml             # app-of-apps
        ├── argocd/               # repo credentials
        ├── apps/                 # child aplikacije
        └── observability/        # Grafana dashboard-i
```
