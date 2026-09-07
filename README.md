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
| 2 | `shophub` | Kreira Shop resurse, pa mu CRD-ovi iz wave 1 moraju već postojati |

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

## Pre prve sinhronizacije

ShopHub potpisuje access token-e ključem iz Secret-a. ArgoCD renderuje chart bez
pristupa klasteru, pa bi generisan ključ bio nov pri svakoj sinhronizaciji i
odjavio bi sve korisnike. Napravi ga jednom:

```bash
kubectl create namespace shophub
kubectl create secret generic shophub-auth -n shophub \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)"
```

## Struktura

```
kube-state
└── clusters/
    └── local/
        ├── cluster.yaml          # metapodaci klastera
        ├── root.yaml             # app-of-apps
        ├── argocd/               # repo credentials
        ├── apps/                 # child aplikacije
        └── observability/        # Grafana dashboard-i
```
