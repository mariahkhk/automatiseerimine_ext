# CI/CD ja Kubernetes

**Eeldused:** Git, Docker, Linux CLI, YAML süntaks

**Platvorm:** GitHub Actions, k3s

**Dokumentatsioon:** [docs.github.com/en/actions](https://docs.github.com/en/actions), [kubernetes.io/docs](https://kubernetes.io/docs)

## Õpiväljundid

- Selgitad CI/CD vajadust ja millist probleemi see lahendab
- Eristad Continuous Integration ja Continuous Deployment kontseptsioone
- Kirjutad GitHub Actions workflow faili
- Selgitad konteinerite orkestreerimise vajadust
- Kirjeldad Kubernetes arhitektuuri ja põhikomponente
- Lood ja haldad Pod'e, Deployment'e ja Service'id kubectl abil

---

## 1. Suur Pilt: Kõik Kokku

### Meie Automatiseerimise Teekond

Selle kursuse jooksul oleme õppinud mitu automatiseerimise tööriista. Iga tööriist lahendas konkreetset probleemi. Git haldab koodi versioone. Docker pakendab rakenduse konteinerisse. Ansible seadistab servereid. Terraform loob infrastruktuuri.

Aga kuidas need kõik koos töötavad? Kes käivitab need protsessid? Kes otsustab millal buildida Docker image? Millal jooksutada Ansible playbook'i? Millal Terraform apply teha? Ja kui Docker konteinereid on sadu, kes haldab neid kõiki?

Vastuseks on kaks viimast tööriista meie kursuses: **CI/CD** ja **Kubernetes**. CI/CD on liim, mis ühendab kõik tööriistad automatiseeritud protsessiks. Kubernetes on dirigent, kes orkestreerib konteinereid suurel skaalal.

### Kuidas Täielik Pipeline Välja Näeb

Arendaja kirjutab koodi ja teeb `git push`. See käivitab GitHub Actions workflow'i. Workflow buildib Docker image'i kasutades Dockerfile'i mida te õppisite. Jooksutab automaatsed testid. Kui testid läbivad, push'ib image Docker Registry'sse. Seejärel Kubernetes deploy'ib selle image'i klasterisse, tagab et alati on õige arv instantse töös, jaotab liiklust nende vahel ja taastab automaatselt krahhinud konteinerid. Kõik see juhtub automaatselt 5-10 minutiga.

Ilma CI/CD ja Kubernetes'eta peaksite te kõik need sammud käsitsi tegema. Iga kord. Iga deployment'iga. See võtaks tunde, oleks vigane ja ei skaleeruks.

```
Developer → git push → GitHub Actions → Docker Build → Test → Push Image
                                                                   ↓
Production ← Kubernetes ← Deploy ← Registry ← Image Ready
```

**Kontrollküsimus:** Millised tööriistad meie kursusest on selles ahelas esindatud?

---

## 2. CI/CD: Miks ja Mis

### Probleem: Käsitsi Deployment

Vaatame konkreetset näidet. Teil on veebirakendus Tallinna startup'is. Te muudate koodi. Kuidas see jõuab production serverisse?

Käsitsi protsess näeb välja nii. Arendaja lõpetab koodi. Testib lokaalselt. Teeb `git push`. Siis logib SSH'iga serverisse. Teeb `git pull`. Installib dependencies käsitsi. Buildib rakenduse. Restartib teenuse. Kontrollib kas töötab. Kui midagi valesti, debugib. Proovib uuesti. See võtab 1-3 tundi. Järgmine kord võib protsess olla natuke erinev, sest inimesed unustavad samme või teevad vigu.

Automaatse CI/CD puhul teeb arendaja `git push` ja 10 minutit hiljem on production'is uus versioon. Sama protsess iga kord. Dokumenteeritud YAML failina. Reprodutseeritav. Usaldusväärne.

| Aspekt | Käsitsi | CI/CD |
|--------|---------|-------|
| **Aeg** | 1-3 tundi | 5-10 minutit |
| **Vead** | Iga kord erinev | Sama protsess |
| **Dokumentatsioon** | Confluence (aegunud) | YAML fail (alati aktuaalne) |
| **Sagedus** | 1x kuus (hirm) | 10-100x päevas |

Kaasaegsed ettevõtted nagu Amazon deploy'ivad 23,000 korda päevas. Netflix 1000 korda päevas. Eestis Bolt ja Wise kasutavad samuti intensiivset CI/CD'd. See on võimalik ainult täieliku automatiseerimisega.

### Integration Hell

Teine probleem mida CI/CD lahendab on "integration hell". Kaks arendajat töötavad eraldi branch'ides kaks nädalat. Mõlemad muudavad koodi, lisavad feature'eid. Siis proovivad merge'ida. Tekivad konfliktid. Kood ei kompileeru. Testid failivad. API'd on muutunud. Kulub päevi et parandada.

CI/CD lahendab selle lihtsalt: integreeri sageli, vähemalt iga päev, ja kontrolli automaatselt. Väikesed muudatused on lihtne debugida. Suur kahenädalane merge on õudusunenägu.

### Continuous Integration vs Continuous Deployment

**Continuous Integration** tähendab et iga kord kui arendaja push'ib koodi, süsteem automaatselt buildib ja testib selle. Developer teeb `git push`. GitHub Actions workflow käivitub. Kood buildib automaatselt. Testid jooksevad automaatselt. Kui midagi ebaõnnestub, developer saab teada 5 minuti jooksul, mitte homme, mitte järgmisel nädalal. Kohe, kui kontekst on veel meeles.

**Continuous Delivery** läheb sammu edasi. Lisaks buildimisele ja testimisele valmistatakse rakendus ette deployment'iks. Inimene vajutab viimase "deploy" nuppu.

**Continuous Deployment** on täiesti automaatne. Kui kood läbib kõik testid ja kvaliteedikontrollid, deploy'itakse automaatselt production'i. Ühtegi inimest vahel pole.

| Aspekt | Continuous Delivery | Continuous Deployment |
|--------|--------------------|-----------------------|
| **Production deploy** | Inimene vajutab nuppu | Täiesti automaatne |
| **Kontroll** | Manual approval | Testide põhjal |
| **Sobib** | Algajad, regulated industries | Mature teams, SaaS |

Enamik ettevõtteid alustab Continuous Delivery'ga ja liigub järk-järgult Continuous Deployment'i poole kui usaldus kasvab.

**Kontrollküsimus:** Miks on enamik ettevõtteid alguses ettevaatlik täisautomaatse deployment'iga?

---

## 3. GitHub Actions

### Miks GitHub Actions?

CI/CD platvorme on mitu. Jenkins on vanim ja enim kasutatud, kuid keerulise seadistusega. GitLab CI on võimas täielik DevOps platvorm. CircleCI on kiire aga kallis. Me õpime GitHub Actions'i, sest see on sisse-ehitatud GitHubi. Kui teil on GitHub repository, teil on GitHub Actions. Zero setup. Ei pea midagi installima ega servereid üles seadma.

GitHub Actions on tasuta avalikele projektidele ja piisavalt tasuta privaatsetele repodele (2000 minutit kuus). Dokumentatsioon on hea, GitHub Marketplace'is on üle 13,000 valmis action'i mida saab koheselt kasutada. Ja kõige olulisem - see on lihtne YAML konfiguratsioon, mida saab versioonihalda koos koodiga.

### Kuidas See Töötab

Kui te push'ite koodi GitHubi ja teil on workflow defineeritud, toimub järgmine. GitHub detekteerib et repositooriumis on `.github/workflows/` kaustas YAML fail. Loob virtuaalserveri (Ubuntu, Windows või macOS). Jooksutab kõik job'id ja step'id selles serveris. Raporteerib tulemused GitHubi Actions tab'is. Kustutab serveri. Te ei pea muretsema serveri haldamise pärast, GitHub teeb kõik ise.

### Neli Põhikontseptsiooni

GitHub Actions koosneb neljast komponendist mida on oluline eristada.

| Komponent | Mis see on | Näide |
|-----------|-----------|-------|
| **Workflow** | Terve protsess, üks YAML fail | `ci.yml` |
| **Job** | Suur tükk tööd, jookseb eraldi serveris | `build`, `test`, `deploy` |
| **Step** | Üks väike operatsioon job'i sees | "Checkout code", "Run tests" |
| **Action** | Taaskasutav komponent (kellegi teise kirjutatud) | `actions/checkout@v4` |

**Workflow** on kirjeldatud YAML failina `.github/workflows/` kaustas. Iga fail on üks workflow. Repositooriumis võib olla mitu workflow'i, näiteks `ci.yml` testideks ja `deploy.yml` deployment'iks. Workflow defineerib millal see käivitub ja mida see teeb.

**Job** on üks suur tükk tööd. Oluline on mõista et iga job jookseb eraldi runner'is ehk eraldi virtuaalserveris. Job'id ei jaga faile ega state'i automaatselt. Job'id võivad jooksda paralleelselt või järjestikku kui määrate `needs:` sõltuvuse.

**Step** on üks konkreetne operatsioon job'i sees. Step'id jooksevad alati järjestikku. Kui üks step ebaõnnestub, järgmised ei käivitu. Step võib olla kas valmis action (`uses:`) või käsurea käsk (`run:`).

**Action** on taaskasutav komponent. Mõelge sellele nagu funktsioonile. Keegi on kirjutanud, testinud, dokumenteerinud ja üles laadinud GitHub Marketplace'i. Sina lihtsalt kasutad. Näiteks `actions/checkout@v4` laeb koodi alla, `actions/setup-python@v5` installib Python'i, `docker/build-push-action@v5` buildib ja push'ib Docker image'i.

**Kontrollküsimus:** Mis vahe on job'il ja step'il? Mis juhtub kui üks step ebaõnnestub?

### Workflow Näide

Vaatame konkreetset workflow faili mis testib Python rakendust ja buildib Docker image'i.

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - run: pip install -r requirements.txt

      - run: pytest tests/ -v

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
```

Selles workflow's on kaks job'i. Esimene job `test` jookseb iga push'i ja pull request'i peale. See laeb koodi alla, installib Python'i, installib dependencies ja jooksutab testid. Teine job `build` jookseb ainult kui testid läbisid (`needs: test`) ja ainult main branch'is (`if: github.ref == 'refs/heads/main'`). See logib Docker Hub'i sisse, buildib Docker image'i ja push'ib selle registry'sse.

`${{ secrets.DOCKER_USERNAME }}` viitab GitHub Secrets'ile mida saab seadistada repo Settings → Secrets → Actions all. Paroolid ja API key'd ei tohi kunagi olla otse YAML failis.

`${{ github.sha }}` on commit hash mis tagab et iga build saab unikaalse tag'i. See on oluline, sest `latest` tag ei ütle midagi konkreetset - te ei tea mis versioon seal tegelikult on.

### Pipeline Loogika

Oluline on mõista et kui üks samm ebaõnnestub, järgmised ei käivitu. Kui build failib, teste ei jooksutata. Kui testid failivad, deployment'i ei toimu. See tagab et ainult töötav kood jõuab kaugemale.

GitHubis iga repositooriumi üleval on "Actions" tab. Seal näete kõiki workflow'e: millised jooksevad (kollane ring), millised õnnestusid (roheline linnuke), millised ebaõnnestusid (punane X). Kui klõpsate workflow'i peale, näete iga job'i ja step'i detailset väljundit ja loge. Pull request'i juures näete automaatselt check'e ja kui branch protection on seadistatud, ei saa merge'ida enne kui kõik check'id on läbitud.

**Kontrollküsimus:** Kus peab workflow fail repositooriumis asuma? Mis juhtub kui testid failivad?

---

## 4. Miks Me Vajame Kubernetes'e?

### Docker Compose Piirid

Olete õppinud Docker'it ja Docker Compose'i. Üks `docker-compose.yml` fail, `docker-compose up` käsk ja teie rakendus töötab. See on suurepärane väikeste projektide jaoks, paar konteinerit ühel masinal.

Kuid mis juhtub kui teie ettevõte kasvab? Tallinna startup algab kahe konteineriga (veebileht + andmebaas), kuid aasta pärast on olukord hoopis teine. Te vajate 50 mikroteenust mis peavad töötama 10 serveril. Iga mikroteenust tuleb eraldi uuendada, skaleerida ja monitoorida. Docker Compose ei oska mitut serverit korraga hallata, sest Compose töötab ainult ühel masinal. Docker Compose ei oska automaatselt taastada krahhinud konteinereid, sest kui konteiner kukub, jääb ta lihtsalt seisma. Docker Compose ei oska liiklust jaotada mitme instantsi vahel ega järk-järgult uuendada ilma downtime'ita.

| Probleem | Docker Compose | Kubernetes |
|----------|----------------|------------|
| Mitu serverit | Ainult 1 masin | Kümned-sajad |
| Auto-taastamine | Ei | Automaatne |
| Load balancing | Käsitsi (nginx) | Sisse-ehitatud |
| Zero-downtime update | Ei (downtime) | Rolling update |
| Skaaleerimine | Käsitsi | Automaatne |

### Bolt Näide

Bolt käivitab sadu mikroteenuseid kümnetes riikides. 2016. aastal kasutasid nad Docker Compose'i ja Bash skripte. Iga deploy võttis tunde, sest DevOps insenerid pidid käsitsi SSH'ga serveritesse logima ja konteinereid restartima. Iga serveri rike tähendas käsitsi parandamist, keegi pidi kell 3 öösel üles tõusma.

2017. aastal liikusid nad Kubernetes'ele. Deploy aeg langes tundidelt 10 minutile. Kubernetes hakkas automaatselt taastama krahhinud konteinereid. Skaaleerimine muutus automaatseks: kui liiklus kasvab, lisab Kubernetes ise instantse juurde.

### Mis on Orkestreerimine?

Kujutage ette 50-liikmelise orkestrit. Ilma dirigendita on kaos, keegi ei tea millal alustada, millist tempot hoida, millal vaiksem olla. Kubernetes on teie konteinerite dirigent.

Orkestreerimine tähendab nelja asja. **Planeerimine** ehk otsustamine millisele serverile konteiner paigutada, vaadates kus on vaba CPU ja RAM. **Tervise jälgimine** ehk pidev kontroll kas konteinerid töötavad, ja kui mitte, automaatne restart. **Skaaleerimine** ehk instantside lisamine kui liiklus kasvab ja eemaldamine kui väheneb. **Uuendamine** ehk uue versiooni deploy järk-järgult, ilma downtime'ita.

**Kontrollküsimus:** Miks Docker Compose ei piisa kui teil on 10 serverit ja 50 teenust?

---

## 5. Kubernetes Arhitektuur

### Klastri Struktuur

Kubernetes klaster koosneb kahest osast: **Control Plane** (juhtimistasand) ja **Worker Nodes** (töötajad).

```
┌─────────────────────── CONTROL PLANE ─────────────────────────┐
│                                                               │
│  API Server        Scheduler       Controller       etcd      │
│  (keskne           (paigutab       Manager          (klastri  │
│   suhtluspunkt)     Pod'e)         (self-healing)    DB)      │
│                                                               │
└───────────────────────────┬───────────────────────────────────┘
                            │
              kubectl ──────┘
                            │
┌───────────────────────────┴───────────────────────────────────┐
│                      WORKER NODES                             │
│                                                               │
│  Node 1                          Node 2                       │
│  ┌──────────────────┐            ┌──────────────────┐         │
│  │ kubelet          │            │ kubelet          │         │
│  │ kube-proxy       │            │ kube-proxy       │         │
│  │ [Pod] [Pod] [Pod]│            │ [Pod] [Pod]      │         │
│  └──────────────────┘            └──────────────────┘         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Control Plane - Aju

Control Plane on klastri "aju" kus tehakse otsuseid. See ei käita konteinereid ise, vaid koordineerib kõike.

**API Server** on Kubernetes'e keskne suhtluspunkt. Kõik käsud, olgu need kubectl'ist, dashboardist või teistest komponentidest, käivad läbi API Serveri. See valideerib päringuid, kontrollib õiguseid ja salvestab muudatused andmebaasi.

**etcd** on hajutatud võti-väärtus andmebaas mis hoiab kogu klastri olekut. Iga Deployment, Service, Pod - kõik on etcd's. Kui etcd kaob, kaob klastri "mälu". Seepärast on backup kriitilise tähtsusega.

**Scheduler** otsustab millisele Node'ile uus Pod paigutada. See vaatab millel node'il on piisavalt CPU ja RAM-i, kas on spetsiaalseid nõudeid, ja teeb aruka valiku. Scheduler ei käivita Pod'i ise, see ainult määrab asukoha.

**Controller Manager** jooksutab kontrollereid mis pidevalt jälgivad klastri olekut. Deployment Controller tagab et õige arv Pod'e töötab. Node Controller jälgib kas node'id on elus. Need kontrollerid töötavad lõputus tsüklis, võrreldes soovitud olekut reaalsusega ja tehes vajalikke muudatusi.

### Worker Nodes - Töötajad

Worker Node'id käitavad tegelikult konteinereid. Igal node'il töötavad kolm peamist komponenti.

**Kubelet** on "agent" igal node'il. See küsib API serverilt milliseid Pod'e peaks käitama ja tagab et need Pod'id töötavad. Kui konteiner kukub, proovib kubelet seda restartida. Kubelet saadab regulaarselt olekuinfot API serverisse.

**Kube-proxy** haldab võrgureegleid et Pod'id saaksid omavahel suhelda isegi kui nad on erinevatel node'idel. See programmeerib iptables reegleid et liiklus jõuaks õigesse kohta.

**Container Runtime** (containerd, CRI-O) käitab konteinereid. Kubernetes ise ei käita konteinereid otse, see kasutab runtime'i.

**Kontrollküsimus:** Mis vahe on Control Plane'il ja Worker Node'il? Miks on etcd backup kriitiline?

---

## 6. Kubernetes Põhikontseptsioonid

### Pod - Väikseim Ühik

Pod on Kubernetes'e väikseim juurutatav ühik. See ei ole konteiner, see on ümbris ühele või mitmele konteinerile. Tavaliselt on Pod'is üks konteiner. Pod'i sees jagavad kõik konteinerid sama võrku ja IP aadressi.

Pod on ajutine. Kui Pod kustub, kaovad andmed (kui pole eraldi volume'd). IP aadress muutub iga restart'iga. Seepärast ei haldagi me Pod'e otse, vaid kasutame Deployment'e mis tagavad et alati on õige arv Pod'e töös.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

Selles YAML failis on `apiVersion: v1` sest Pod on põhiressurss. `metadata.labels` on võti-väärtus paarid mida Service kasutab Pod'ide leidmiseks. `spec.containers` defineerib konteinerid mis Pod'is jooksevad.

### Deployment - Deklaratiivne Haldus

Deployment on see mida te päriselt kasutate. Te ütlete "tahan 3 koopiat oma rakendusest" ja Kubernetes tagab et need 3 koopiat alati töötavad. Kui üks Pod kukub, loob Deployment automaatselt uue. See on **self-healing**.

```
Deployment (soovitud: 3 koopiat)
    └── ReplicaSet (haldab Pod'e)
            ├── Pod 1 (nginx:1.25)
            ├── Pod 2 (nginx:1.25)
            └── Pod 3 (nginx:1.25)   ← kukub!
                                      ← Deployment loob uue automaatselt
```

Kui uuendate rakendust, teeb Deployment **rolling update** - loob järk-järgult uued Pod'id enne vanade kustutamist. Kunagi ei ole kõik Pod'id korraga maas, seega kasutajad ei näe downtime'i.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

Siin on `apiVersion: apps/v1` sest Deployment on kõrgema taseme ressurss. `replicas: 3` ütleb mitu Pod'i peab alati töötama. `selector.matchLabels` peab **täpselt** klapima `template.metadata.labels`'iga - see on levinuim viga algajatel. `template` on sisuliselt Pod'i YAML nested Deployment'i sisse.

### Service - Püsiv Võrguaadress

Pod'idel on IP aadressid, kuid need muutuvad iga restart'iga. Service lahendab selle probleemi andes püsiva IP aadressi ja DNS nime. Service leiab Pod'id label'ite järgi ja teeb automaatselt load balancing'u, jaotades päringuid kõigi tervete Pod'ide vahel.

```
Klient → nginx-service (püsiv DNS/IP) → Pod 1
                                       → Pod 2  (round-robin)
                                       → Pod 3
```

**ClusterIP** on vaikimisi Service tüüp - sisemine IP, kättesaadav ainult klastri seest. **NodePort** avab pordi kõigil node'idel vahemikus 30000-32767, võimaldades välist ligipääsu. **LoadBalancer** loob cloud provider'i load balancer'i.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
  type: NodePort
```

`selector` leiab Pod'e dünaamiliselt. Kui uus Pod ilmub label'iga `app: nginx`, lisatakse ta automaatselt Service'i taha. `port` on Service'i port, `targetPort` on Pod'i port (need võivad olla erinevad), `nodePort` on väline port Node'il. DNS nimi `nginx-service` töötab klastri sees automaatselt.

**Kontrollküsimus:** Miks me ei kasuta Pod'i IP'd otse? Miks on Service vajalik?

---

## 7. kubectl ja k3s

### kubectl - Peamine Tööriist

kubectl on Kubernetes'e käsurea tööriist mis räägib API Serveriga. Sellega loote, uuendate, kustutate ja jälgite ressursse.

Põhikäsud mida vajate iga päev. `kubectl get pods` näitab kõiki Pod'e. `kubectl describe pod <nimi>` annab detailse info, sealhulgas event'id mis on debugging'uks kriitilised. `kubectl logs <pod>` näitab Pod'i loge. `kubectl exec -it <pod> -- sh` logib Pod'i sisse nagu SSH.

`kubectl apply -f deployment.yaml` on deklaratiivne lähenemine - te kirjeldate soovitud olekut YAML failis ja Kubernetes teeb selle reaalsuseks. See käsk on idempotent, te võite käivitada seda mitu korda ja tulemus on sama.

Skaleerimine on lihtne: `kubectl scale deployment nginx --replicas=5`. Rolling update samuti: `kubectl set image deployment/nginx nginx=nginx:1.26`. Ja kui uus versioon osutub vigaseks, rollback ühe käsuga: `kubectl rollout undo deployment/nginx`.

### k3s - Kerge Kubernetes

Laboris kasutame k3s'i mis on kerge Kubernetes distributsioon. See on ainult ~100MB (võrreldes täis-Kubernetes'e ~1GB-ga), paigaldub ühe käsuga ja on 100% API compatible. Kõik mis õpite k3s'iga, töötab täis-Kubernetes'es. Erinevus on ainult suuruses ja paigaldamise lihtsuses, kontseptsioonid ja käsud on täpselt samad.

---

## 8. Kõik Koos: CI/CD + Kubernetes

### Täielik Pipeline

Nüüd ühendame kõik. Arendaja kirjutab koodi ja teeb `git push`. GitHub Actions käivitab workflow'i mis jooksutab testid ja buildib Docker image'i. Kui kõik läbib, push'ib image Docker Hub'i. Seejärel GitHub Actions käivitab deployment'i käsu mis uuendab Kubernetes Deployment'i uue image versiooniga. Kubernetes teeb rolling update, asendades vanu Pod'e uutega järk-järgult. Kasutajad ei näe downtime'i.

```yaml
name: Full Pipeline

on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest tests/ -v

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          kubectl set image deployment/myapp \
            myapp=${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
```

Kolm job'i, kolm etappi. Test → Build → Deploy. Kui test failib, build ei käivitu. Kui build failib, deploy ei käivitu. Ainult töötav, testitud kood jõuab production'i.

### Wise Näide

Wise käitab rahvusvahelist maksesüsteemi kus downtime maksab kümneid tuhandeid eurosid minutis. Nende pipeline näeb välja sarnaselt: arendaja push'ib koodi, automaatsed testid jooksevad, Docker image buildib, Kubernetes deploy'ib. 50+ deploy'd päevas ilma teenuse katkestusteta. See on CI/CD + Kubernetes koos töötamas.

---

## Kokkuvõte

### CI/CD

CI/CD ühendab kõik automatiseerimise tööriistad üheks automatiseeritud protsessiks. Käsitsi deployment mis võtab tunde muutub automaatseks protsessiks mis võtab minuteid. GitHub Actions koosneb neljast komponendist: Workflow (YAML fail), Job (eraldi server), Step (üks operatsioon), Action (taaskasutav komponent). Pipeline tagab et ainult testitud kood jõuab production'i.

### Kubernetes

Kubernetes lahendab konteinerite haldamise probleemi suurel skaalal. Pod on väikseim ühik, konteinerite ümbris. Deployment haldab Pod'e ja tagab et alati on õige arv töös, self-healing ja rolling updates. Service annab püsiva võrguaadressi ja teeb load balancing'u. kubectl on CLI tööriist millega seda kõike hallatakse.

Koos moodustavad CI/CD ja Kubernetes täieliku automatiseeritud pipeline'i: koodist production'i ilma käsitsi sekkumiseta.

---

## Ressursid

**CI/CD:**

- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [GitHub Skills](https://skills.github.com/)

**Kubernetes:**

- [Kubernetes Docs](https://kubernetes.io/docs/home/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [k3s Documentation](https://docs.k3s.io/)

---

**Järgmine tund:** Labor - loome CI/CD pipeline'i ja paigaldame k3s klastri.
