# CI/CD ja Kubernetes Labor

**Eeldused:** Git, Docker, GitHub konto, Linux CLI

**Platvorm:** GitHub Actions, k3s (Proxmox VM, Ubuntu 22.04)

**Oluline:** See labor sisaldab tahtlikke vigu! Sinu ülesanne on need leida ja parandada.

## Õpiväljundid

- Loob GitHub Actions workflow faili ja seadistab CI pipeline'i
- Debugib pipeline vigu logide abil
- Paigaldab k3s Kubernetes klastri
- Loob ja haldab Deployment'e ja Service'id
- Testib self-healing ja rolling update funktsionaalsust

---

## Enne Kui Alustad

See labor võtab umbes 3 tundi (4x45 min). Esimene pool keskendub CI/CD pipeline'i loomisele GitHub Actions'iga, teine pool Kubernetes klastri üles seadmisele ja rakenduste deploy'imisele. Lab'i lõpuks on sul töötav automatiseeritud süsteem mis testib koodi ja deploy'ib konteinereid.

Selles lab'is on tahtlikke vigu. Need on seal õppimise eesmärgil. Sinu ülesanne on järgida juhiseid, push'ida koodi GitHubi, vaadata mis GitHub Actions ütleb, leida ja parandada vead, push'ida uuesti. See on päris maailm - CI/CD leiab vigu ja sina pead need parandama.

!!! tip "Lab'i reegel"
    Kui midagi ebaõnnestub, ära arva. Loe alati: (1) error message, (2) mis käsk ebaõnnestus, (3) mida süsteem ootas vs mida sai. See on sama debugging loogika mida kasutad igapäevatöös.

---

## 1. Rakenduse ja Git Setup

### Projekti Loomine

Alustame lihtsa Flask rakenduse loomisega mida kasutame kogu lab'i jooksul.

```bash
mkdir cicd-k8s-lab                 # Projekti kaust
cd cicd-k8s-lab
git init                           # Initsialiseerib Git repo
git branch -M main                 # Nimetab vaikebranch'i 'main' (GitHub standard)
```

### Flask Rakendus

Loo fail nimega `app.py`. See on lihtne REST API kolme endpointiga: pealeht, health check ja toodete list.

```python
from flask import Flask, jsonify       # Flask veebiraamistik, jsonify teeb dict -> JSON
from datetime import datetime          # Ajatempli jaoks

app = Flask(__name__)                  # Loob Flask rakenduse

@app.route('/')                        # Pealeht: GET /
def home():
    return jsonify({
        'message': 'CI/CD + K8s Demo API',
        'version': '1.0.0',           # Versioon - testid kontrollivad seda!
        'timestamp': str(datetime.now())
    })

@app.route('/health')                  # Health check: GET /health
def health():                          # K8s ja CI pipeline kasutavad seda
    return jsonify({'status': 'healthy'}), 200  # 200 = OK

@app.route('/products')                # Toodete nimekiri: GET /products
def products():
    return jsonify([
        {'id': 1, 'name': 'Laptop', 'price': 999},
        {'id': 2, 'name': 'Phone', 'price': 599}   # Seda hinda muudame hiljem testiks
    ])

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000) # 0.0.0.0 = kuulab kõigil IP'del (vajalik Docker'is)
```

Loo fail nimega `requirements.txt`:

```
Flask==3.0.0
pytest==8.0.0
```

### Kohalik Testimine

Enne kui läheme CI/CD juurde, veenduge et rakendus töötab kohalikult.

```bash
pip install -r requirements.txt
python app.py
```

Avage teine terminal ja testige:

```bash
curl http://localhost:5000/
curl http://localhost:5000/health
curl http://localhost:5000/products
```

Peaksite nägema JSON vastuseid. Peatage rakendus Ctrl+C'ga.

### Git Repository

```bash
echo "__pycache__/" > .gitignore   # Python kompileeritud failid (ei kuulu Git'i)
echo "*.pyc" >> .gitignore         # Python bytecode failid
echo "venv/" >> .gitignore         # Virtual environment (kui kasutad)

git add .
git commit -m "Initial: Flask API"
```

Mine GitHub'i ja loo uus repository nimega `cicd-k8s-lab` (Public). Seejärel ühenda oma kohalik repo GitHubiga:

```bash
git remote add origin https://github.com/SINU_USERNAME/cicd-k8s-lab.git
git push -u origin main
```

**Validation:**

- [ ] Flask app vastab kohalikult kõigil kolmel endpointil
- [ ] Kood on GitHub'is nähtav

---

## 2. CI Pipeline: Validate ja Test

### Workflow Faili Loomine

Loo kataloog ja fail `.github/workflows/ci.yml`. See on meie pipeline'i süda - GitHub Actions loeb selle faili ja teab mida teha.

```bash
mkdir -p .github/workflows
```

Loo fail `.github/workflows/ci.yml`:

```yaml
name: CI/CD Pipeline              # Workflow nimi (nähtav GitHub Actions UI's)

on:                                # Millal käivitub?
  push:
    branches: [ main ]             # Iga push main branch'i
  pull_request:
    branches: [ main ]             # Iga PR main branch'i vastu

jobs:
  validate:                        # Esimene job: süntaksi kontroll
    runs-on: ubuntu-latest         # Jookseb GitHub'i Ubuntu VM'is
    steps:
      - uses: actions/checkout@v4  # Kloonib meie repo

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'   # Sama versioon mis Dockerfile'is

      - name: Validate Python syntax
        run: python -m py_compile app.py  # Kompileerib, ei käivita - leiab süntaksivead
```

See workflow käivitub iga kord kui push'ite main branch'i. Esimene job `validate` kontrollib et Python süntaks on korrektne. See on kiireim viis vigu leida, enne kui hakkame dependencies installima ja teste jooksutama.

### Push ja Vaata

```bash
git add .github/
git commit -m "Add pipeline: validate stage"
git push origin main
```

Mine GitHub'is Actions tab'i alla. Pipeline peaks olema roheline. Kui näete kollast ringi, siis workflow veel jookseb. Oodake kuni lõpeb.

### Ülesanne 2.1: Süntaksi Viga

Nüüd teeme tahtliku vea et näha kuidas pipeline reageerib.

Muuda `app.py` real kus on `def home():` - eemalda koolon lõpust nii et jääb `def home()`. Push'i GitHubi:

```bash
git add app.py
git commit -m "Test: intentional syntax error"
git push origin main
```

Mine GitHub Actions'i. Näete punast X'i. Klõpsake pipeline'i peale ja lugege error message'it. See ütleb täpselt mis real viga on. Parandage viga (lisage koolon tagasi) ja push'ige uuesti. Pipeline peaks olema jälle roheline.

**Mõtle:** Kui kiiresti said teada et midagi oli valesti? See on validate stage'i väärtus. Ilma CI/CD'ta oleks see viga avastatud alles siis kui keegi proovib rakendust käivitada.

### Testide Lisamine

Loo fail nimega `test_app.py`. Need testid kontrollivad et meie API endpointid töötavad õigesti ja et äriloogika on korrektne.

```python
import pytest
from app import app                    # Impordime meie Flask rakenduse

@pytest.fixture                        # Fixture = korduvkasutatav test setup
def client():
    app.config['TESTING'] = True       # Lülitab Flask'i test režiimi
    with app.test_client() as client:  # Loob test kliendi (ei käivita päris serverit)
        yield client                   # yield = annab kliendi testile, pärast koristab

def test_health(client):
    response = client.get('/health')   # Simuleerib GET /health päringut
    assert response.status_code == 200
    assert response.get_json()['status'] == 'healthy'

def test_home(client):
    response = client.get('/')
    assert response.status_code == 200
    data = response.get_json()
    assert data['version'] == '1.0.0'  # Kontrollib et versioon klapib
    assert 'message' in data           # Kontrollib et väli eksisteerib

def test_products(client):
    response = client.get('/products')
    assert response.status_code == 200
    products = response.get_json()
    assert len(products) == 2          # Ootame täpselt 2 toodet

    for product in products:           # Ärireegli kontroll: hinnad peavad olema positiivsed
        assert product['price'] > 0, f"Hind peab olema positiivne! Leitud: {product['price']}"
```

Viimane test kontrollib ärireegleid - hinnad peavad olema positiivsed. See on oluline erinevus süntaksi kontrollist. Validate leiab kirjavead, testid leiavad loogikavead.

Testige kohalikult:

```bash
pytest test_app.py -v
```

Kõik testid peaksid läbima.

### Test Stage Lisamine

Uuendage `.github/workflows/ci.yml` et lisada test job:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Validate Python syntax
        run: python -m py_compile app.py

  test:
    needs: validate                # Jookseb AINULT kui validate läbib
    runs-on: ubuntu-latest         # Iga job saab puhtalt uue VM'i
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'             # Vahemälustab pip pakette (kiirendab)

      - name: Install dependencies
        run: pip install -r requirements.txt  # Flask + pytest

      - name: Run tests
        run: pytest test_app.py -v  # -v = verbose, näitab iga testi tulemust
```

Pange tähele `needs: validate` - test job jookseb ainult kui validate on läbitud. Push'ige ja kontrollige Actions tab'is et mõlemad job'id on rohelised.

```bash
git add test_app.py .github/workflows/ci.yml
git commit -m "Add tests and test stage"
git push origin main
```

### Ülesanne 2.2: Negatiivne Hind

Nüüd testime äriloogika kontrolli. Muuda `app.py` products funktsioonis Phone hind: `'price': 599` muutke `'price': -599`. Push'ige GitHubi.

```bash
git add app.py
git commit -m "Bug: negative price"
git push origin main
```

Vaadake GitHub Actions'is: Validate on roheline (süntaks on õige!) aga Test on punane (äriloogika on vale!). Lugege error message'it - test ütleb täpselt mis on valesti. Parandage hind positiivseks ja push'ige uuesti.

**Mõtle:** Miks validate ei leidnud seda viga aga test leidis? Validate kontrollib ainult süntaksit. Test kontrollib kas rakendus käitub õigesti.

!!! note "Mentaalne mudel"
    **Validate** vastab küsimusele: "Kas see kood on õigesti kirjutatud?"
    **Test** vastab küsimusele: "Kas see kood käitub õigesti?"

**Validation:**

- [ ] Validate stage leidis süntaksi vea (ülesanne 2.1)
- [ ] Test stage leidis negatiivse hinna (ülesanne 2.2)
- [ ] Mõistad erinevust validate ja test vahel

---

## 3. Docker Build Stage

### Dockerfile

Loo fail nimega `Dockerfile`. Selles on tahtlik viga mille peate ise leidma.

```dockerfile
FROM python:3.11-slim              # Baas-image: Python 3.11 (slim = väiksem, ilma arendusvahenditeta)
WORKDIR /app                       # Töökataloog konteineris (luuakse automaatselt)

COPY requirements.txt .            # Kopeerime esmalt ainult requirements (Docker cache optimeerimine)
RUN pip install --no-cache-dir -r requirements.txt  # --no-cache-dir = väiksem image

COPY app.py .                      # Kopeerime rakenduse koodi

EXPOSE 5000                        # Dokumenteerib et rakendus kuulab port 5000

CMD ["python", "main.py"]          # TAHTLIK VIGA: main.py ei eksisteeri, peab olema app.py
```

Testi kohalikult:

```bash
docker build -t cicd-k8s-lab:test .               # Ehitab image'i (. = praegune kaust)
docker run -d -p 5000:5000 --name test-app cicd-k8s-lab:test  # -d = taustal, -p = port mapping
curl http://localhost:5000/health                  # Testib kas rakendus vastab
docker stop test-app && docker rm test-app         # Koristab
```

### Build Stage Lisamine

Lisa `ci.yml` faili uus job. See buildib Docker image'i ja push'ib selle GitHub Container Registry'sse (ghcr.io) mis on tasuta ja integreeritud GitHubiga.

```yaml
  build:
    needs: test                    # Jookseb ainult kui testid läbivad
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'  # Ainult main branch (mitte PR'id)
    permissions:                   # Õigused GHCR'i push'imiseks
      contents: read               # Repo sisu lugemine
      packages: write              # Container registry kirjutamine
    steps:
      - uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io        # GitHub'i enda container registry
          username: ${{ github.actor }}      # Automaatne: kes push'is
          password: ${{ secrets.GITHUB_TOKEN }}  # Automaatne: GitHub'i token

      - name: Build Docker image
        run: |
          IMAGE="ghcr.io/${GITHUB_REPOSITORY,,}"  # ,, = lowercase (GHCR nõuab)
          docker build -t $IMAGE:${{ github.sha }} .  # Tag commit hash'iga (unikaalne)
          docker build -t $IMAGE:latest .             # Tag latest (mugavuse jaoks)

      - name: Test Docker image
        run: |
          IMAGE="ghcr.io/${GITHUB_REPOSITORY,,}":${{ github.sha }}
          docker run -d -p 5000:5000 --name test-container $IMAGE  # Käivitab konteineri
          sleep 5                  # Ootab et rakendus jõuaks käivituda

          if curl -f http://localhost:5000/health; then  # -f = fail on HTTP error
            echo "Health check passed"
          else
            echo "Health check FAILED"
            docker logs test-container  # Näitab miks konteiner crashis
            exit 1                     # Lõpetab pipeline punasega
          fi

          docker stop test-container
          docker rm test-container     # Koristab pärast testi

      - name: Push Docker image
        run: |
          IMAGE="ghcr.io/${GITHUB_REPOSITORY,,}"
          docker push $IMAGE:${{ github.sha }}  # Push'ib mõlemad tag'id
          docker push $IMAGE:latest
```

Enne push'imist mine repo Settings → Actions → General → Workflow permissions → **Read and write permissions** → Save. Lisaks on workflow failis `permissions:` blokk mis tagab et build job saab GHCR'i push'ida.

### Ülesanne 3.1: Paranda Dockerfile

!!! note "Oluline Docker kontseptsioon"
    Docker image buildib edukalt isegi kui failinimi CMD's on vale. Build kopeerib failid ja installib dependencies - see ei käivita rakendust. Viga ilmneb alles siis kui konteiner proovib käivituda. Seepärast on CI pipeline'is oluline testida ka konteinerit, mitte ainult buildi.

Push'ige praegune kood:

```bash
git add Dockerfile .github/workflows/ci.yml
git commit -m "Add Docker build stage"
git push origin main
```

Vaadake Actions tab'is. Validate läbib, Test läbib, aga Build? Vaadake logi hoolikalt. Health check test käivitab konteineri ja proovib `curl http://localhost:5000/health`. Konteiner crashib kohe - vaadake Docker logisid mida pipeline väljastab. Mis faili Python üritab käivitada ja mis faili te tegelikult COPY'site?

Parandage Dockerfile nii et CMD käivitab õiget faili. Push'ige uuesti ja kontrollige et Build on roheline.

**Validation:**

- [ ] Dockerfile viga leitud ja parandatud (CMD failinimi)
- [ ] Build stage läbib rohelisena
- [ ] Docker image on GitHub Container Registry's

---

## 4. k3s Setup

### VM Kontrollimine

Nüüd liigume Kubernetes'e juurde. Teie õpetaja on loonud Proxmox VM'i. Logige sisse SSH kaudu.

```bash
ssh student@10.82.1.10
```

Kontrollige süsteemi:

```bash
uname -a                           # Kerneli versioon ja arhitektuur
cat /etc/os-release | grep VERSION # Ubuntu versioon
ping -c 3 8.8.8.8                  # Interneti ühenduse kontroll
docker --version                   # Docker peab olema installitud
```

### k3s Installimine

k3s on kerge Kubernetes distributsioon. Meie laboris jookseb kõik ühes VM'is - nii Control Plane kui Worker. Production'is oleksid need eraldi masinatel, aga kontseptsioonid on täpselt samad.

```bash
curl -sfL https://get.k3s.io | sh -
```

See võtab 1-2 minutit. Pärast installimist seadistage kubectl:

```bash
mkdir -p $HOME/.kube                   # Loob .kube kausta (kui pole olemas)
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config  # k3s config -> kubectl config
sudo chown $USER:$USER $HOME/.kube/config  # Muudab omanikuks praeguse kasutaja
chmod 600 $HOME/.kube/config           # Ainult omanik saab lugeda (turvalisus)
```

Kontrollige et klaster töötab:

```bash
kubectl get nodes
```

Peaksite nägema oma node'i STATUS "Ready". Kui näete "NotReady", oodake 30 sekundit ja proovige uuesti - k3s alles käivitub.

```bash
kubectl get pods -n kube-system
```

See näitab Kubernetes'e enda süsteemikomponente. Kõik peaksid olema Running staatuses.

**Validation:**

- [ ] `kubectl get nodes` näitab STATUS "Ready"
- [ ] `kubectl get pods -n kube-system` näitab Running Pod'e

---

## 5. Pod'id ja Deployment'id

### Esimene Pod

Loome lihtsa nginx Pod'i et mõista kuidas Kubernetes konteinereid käivitab.

```bash
mkdir ~/k8s-lab
cd ~/k8s-lab

kubectl run nginx-test --image=nginx:1.25 --port=80
```

Kui käivitate `kubectl get pods`, näete et Pod on alguses `ContainerCreating` staatuses - Kubernetes tõmbab Docker image'i. 10-30 sekundi pärast peaks olema `Running`.

Uurige Pod'i lähemalt:

```bash
kubectl describe pod nginx-test    # Detailne info + Events (parim debugging käsk)
kubectl logs nginx-test            # Konteineri logid (stdout/stderr)
kubectl exec -it nginx-test -- curl localhost  # Käivitab käsu Pod'i sees (-it = interaktiivne)
```

`describe` näitab Pod'i tervet ajalugu Events sektsioonis - see on debugging'uks kõige kasulikum käsk. `logs` näitab konteineris jooksvaid loge. `exec` logib Pod'i sisse nagu SSH.

Kustutage test Pod:

```bash
kubectl delete pod nginx-test
```

### Deployment YAML

Pod'i otse haldamine pole praktiline - kui ta kukub, on läinud. Deployment tagab et alati on õige arv Pod'e töös. Looge fail `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1                # API versioon (apps/v1 Deployment'ide jaoks)
kind: Deployment                   # Ressursi tüüp
metadata:
  name: nginx-deploy               # Deployment'i nimi (kubectl get deployments)
spec:
  replicas: 3                      # Mitu Pod'i alati töös peab olema
  selector:
    matchLabels:
      app: nginx                   # Selector: milliseid Pod'e see Deployment haldab
  template:                        # Pod'i mall (template) - sellest luuakse Pod'id
    metadata:
      labels:
        app: nginx                 # Label PEAB klapima selector'iga! (levinuim viga)
    spec:
      containers:
        - name: nginx              # Konteineri nimi Pod'i sees
          image: nginx:1.25        # Docker image ja versioon
          ports:
            - containerPort: 80    # Port mida konteiner kuulab
          resources:
            requests:              # Minimaalsed ressursid (K8s garanteerib need)
              memory: "32Mi"
              cpu: "50m"           # 50 millicores = 5% ühest CPU tuumast
            limits:                # Maksimaalsed ressursid (K8s tapab kui ületab)
              memory: "64Mi"
              cpu: "100m"
```

`selector.matchLabels` peab täpselt klapima `template.metadata.labels`'iga. See on levinuim viga algajatel. Kui need ei klapi, Deployment ei leia oma Pod'e ja te saate kummalise errori.

!!! tip "Mõtle nii"
    Labels on nagu nimesildid. Selector on nagu otsingufilter. Kui nimesilt ütleb `app: nginx` aga filter otsib `app: web`, siis Kubernetes ei "näe" Pod'e.

```bash
kubectl apply -f nginx-deployment.yaml

kubectl get deployments
kubectl get pods
```

Deployment lõi automaatselt 3 Pod'i. Pod nimed on formaadis `nginx-deploy-<hash>-<random>`.

### Self-Healing Test

Miks me kustutame Pod'i meelega? Sest production süsteemid kukuvad pidevalt. Serverid jooksevad kokku, võrk katkeb, mälu saab täis. Kubernetes on väärtuslik ainult siis kui ta parandab rikked ilma inimese sekkumiseta. Testime seda.

Kustutage üks Pod ja vaadake mis juhtub.

```bash
kubectl get pods
```

Kopeerige ühe Pod'i nimi ja kustutage see:

```bash
kubectl delete pod nginx-deploy-xxxxx
```

Kohe pärast kustutamist vaadake uuesti:

```bash
kubectl get pods
```

Näete et üks Pod on `Terminating` ja uus on `ContainerCreating`. 10 sekundi pärast on jälle 3 töötavat Pod'i. Deployment märkas et on ainult 2/3 ja lõi automaatselt uue. See on self-healing.

### Skaleerimine

```bash
kubectl scale deployment nginx-deploy --replicas=5  # Suurenda 3 -> 5 (lisab 2 Pod'i)
kubectl get pods

kubectl scale deployment nginx-deploy --replicas=2  # Vähenda 5 -> 2 (kustutab 3 Pod'i)
kubectl get pods
```

Esimesel juhul loodi 2 Pod'i juurde, teisel kustutati 3. Kubernetes hoiab alati täpselt nii palju Pod'e kui te ütlesite.

### Rolling Update

Uuendame nginx versiooni 1.25 → 1.26 ilma downtime'ita.

```bash
kubectl set image deployment/nginx-deploy nginx=nginx:1.26  # Muudab image versiooni

kubectl rollout status deployment/nginx-deploy  # Ootab kuni update on lõppenud
```

Kubernetes loob ühe uue Pod'i versiooniga 1.26, ootab kuni see on valmis, kustutab ühe vana Pod'i versiooniga 1.25, ja kordab kuni kõik on uuendatud. Kasutajad ei näe kunagi downtime'i.

Kui uus versioon ei tööta, saate minna tagasi:

```bash
kubectl rollout undo deployment/nginx-deploy  # Taastab eelmise versiooni (1.25)
```

**Validation:**

- [ ] Deployment lõi 3 Pod'i automaatselt
- [ ] Self-healing töötas (Pod taastus pärast kustutamist)
- [ ] Skaleerimine muutis Pod'ide arvu
- [ ] Rolling update ja rollback töötasid

---

## 6. Service'id

### ClusterIP Service

Pod IP'd muutuvad iga restart'iga, seega vajame püsivat aadressi. Looge fail `nginx-service.yaml`:

```yaml
apiVersion: v1                     # Core API (Service on põhiressurss)
kind: Service
metadata:
  name: nginx-service              # DNS nimi klastris (Pod'id kasutavad seda)
spec:
  selector:
    app: nginx                     # Leiab Pod'id selle label'iga
  ports:
    - port: 80                     # Port millel Service kuulab
      targetPort: 80               # Port kuhu suunab (konteineri port)
  type: ClusterIP                  # Ainult klastri sees kättesaadav (vaikimisi)
```

`selector: app: nginx` leiab kõik Pod'id selle label'iga. Kui Pod ilmub või kaob, uuendab Service automaatselt oma endpoint'ide listi.

```bash
kubectl apply -f nginx-service.yaml    # Loob Service'i YAML'i järgi

kubectl get services                   # Näitab kõiki Service'id (ClusterIP aadress)
kubectl get endpoints nginx-service    # Näitab Pod IP'd kuhu Service suunab
```

Endpoints näitab Pod'ide IP aadresse kuhu Service suunab liiklust.

### DNS Test

Testige et DNS nimi töötab klastri sees:

```bash
kubectl run test-dns --image=busybox --rm -it --restart=Never -- sh
# --rm = kustutab Pod'i pärast väljumist
# -it = interaktiivne terminal
# --restart=Never = ühekordne Pod (mitte Deployment)
```

Busybox'i sees:

```bash
nslookup nginx-service             # Küsib DNS'ilt: mis IP on nginx-service?
wget -q -O- http://nginx-service   # Teeb HTTP päringu Service nimele
exit
```

Kui `nslookup` ei ole busybox'is saadaval, piisab `wget` reast - see tõestab et DNS töötab.

DNS nimi `nginx-service` lahendub automaatselt Service'i IP aadressiks. See on Kubernetes'e sisseehitatud DNS.

!!! note "Oluline kontseptsioon"
    Kubernetes loob automaatselt DNS kirje iga Service jaoks. Pod'id ei räägi kunagi IP aadressidega - nad räägivad teenuse nimedega. See tähendab et kui Pod restartib ja saab uue IP, ei pea midagi muutma.

### NodePort - Väline Ligipääs

ClusterIP on ainult klastri sees. Et pääseda ligi väljastpoolt, vajame NodePort'i. Looge fail `nginx-nodeport.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx                     # Sama selector mis ClusterIP'l
  ports:
    - port: 80                     # Service'i port (klastri sees)
      targetPort: 80               # Konteineri port
      nodePort: 30080              # Väline port (30000-32767 vahemik)
  type: NodePort                   # Avab pordi node'i IP'l (väljast kättesaadav)
```

```bash
kubectl apply -f nginx-nodeport.yaml
```

Nüüd peaksite saama avada brauseris `http://10.82.1.10:30080` (asendage oma VM'i IP'ga) ja näha nginx welcome lehte.

Liikluse voog on: brauser → Node port 30080 → Service → Pod port 80. Kolm kihti, kuid kasutaja näeb ainult esimest.

### PostgreSQL Rakendus

Loome kahekihilise rakenduse et näha kuidas teenused omavahel suhtlevad. Looge fail `postgres.yaml`:

```yaml
# --- RESSURSS 1: Secret (parool) ---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque                       # Opaque = üldine Secret tüüp
data:
  password: cGFzc3dvcmQxMjM=      # "password123" base64 kodeeritult (echo -n password123 | base64)
---
# --- RESSURSS 2: Deployment (andmebaas) ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1                      # Andmebaas = alati 1 replika (andmed!)
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14-alpine  # Alpine = väiksem image
          env:                       # Keskkonnamuutujad (PostgreSQL seadistus)
            - name: POSTGRES_DB
              value: "testdb"        # Loob selle andmebaasi automaatselt
            - name: POSTGRES_USER
              value: "testuser"
            - name: POSTGRES_PASSWORD
              valueFrom:             # Parool tuleb Secret'ist, mitte otse YAML'ist
                secretKeyRef:
                  name: postgres-secret  # Viitab ülal loodud Secret'ile
                  key: password          # Võtme nimi Secret'is
          ports:
            - containerPort: 5432    # PostgreSQL vaikeport
          resources:
            requests:                # Minimaalsed garanteeritud ressursid
              memory: "128Mi"
              cpu: "100m"
            limits:                  # Maksimaalsed lubatud ressursid
              memory: "256Mi"
              cpu: "200m"
---
# --- RESSURSS 3: Service (võrguühendus) ---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service           # DNS nimi: teised Pod'id ühenduvad siia
spec:
  selector:
    app: postgres                  # Leiab PostgreSQL Pod'i
  ports:
    - port: 5432                   # Service kuulab samal pordil
      targetPort: 5432             # Suunab konteineri porti
  type: ClusterIP                  # Ainult klastri sees (andmebaas ei tohi olla väljas!)
```

See YAML sisaldab kolme ressurssi ühes failis, eraldatud `---` märgiga. Secret hoiab parooli base64 kodeeritult (see ei ole krüpteerimine, ainult kodeerimine). Deployment kasutab Secret'ist parooli `valueFrom.secretKeyRef` abil, nii et parool ei ole otse YAML failis nähtav. Service on ClusterIP tüüpi, sest andmebaas ei tohi olla väljastpoolt kättesaadav.

!!! warning "Base64 ≠ krüpteerimine"
    Base64 kaitseb juhusliku nähtavuse eest (parool ei paista YAML'ist välja), aga igaüks kellel on klastri ligipääs saab Secret'i dekodeerida ühe käsuga. Production'is kasuta Sealed Secrets või HashiCorp Vault.

```bash
kubectl apply -f postgres.yaml

kubectl get pods -l app=postgres
```

Oodake kuni Pod on Running. Seejärel testige ühendust:

```bash
kubectl run psql-test --image=postgres:14-alpine --rm -it --restart=Never -- \
  sh -lc 'PGPASSWORD=password123 psql -h postgres-service -U testuser -d testdb'
# -h postgres-service = ühendub DNS nime kaudu (mitte IP)
# PGPASSWORD = parool käsurealt (vältimaks interaktiivset küsimust)
```

PostgreSQL shell'is saate käivitada `\l` et näha andmebaase ja `\q` et väljuda. DNS nimi `postgres-service` suunas meid automaatselt õigesse Pod'i.

**Validation:**

- [ ] ClusterIP Service loodud ja DNS töötab
- [ ] NodePort Service kättesaadav brauserist
- [ ] PostgreSQL käivitus ja Service suunas ühenduse õigesse Pod'i

---

## 7. Cleanup

```bash
kubectl delete deployment nginx-deploy postgres  # Kustutab Deployment'id (ja nende Pod'id)
kubectl delete service nginx-service nginx-nodeport postgres-service  # Kustutab Service'id
kubectl delete secret postgres-secret            # Kustutab Secret'i

kubectl get all                    # Kontrolli et kõik on puhas
```

Pärast cleanup'i peaks olema ainult `kubernetes` Service.

---

## Kontrollnimekiri

**CI/CD tagasiside:**

- [ ] Validate stage leidis süntaksi vea
- [ ] Test stage leidis negatiivse hinna
- [ ] Mõistad erinevust validate ja test vahel

**Kontaineriseerimine:**

- [ ] Dockerfile viga leitud ja parandatud (CMD failinimi)
- [ ] Build stage läbib rohelisena
- [ ] Docker image on GitHub Container Registry's

**Orkestreerimine:**

- [ ] k3s paigaldatud ja töötab
- [ ] Deployment lõi 3 Pod'i
- [ ] Self-healing töötas (Pod taastus pärast kustutamist)
- [ ] Rolling update ja rollback töötasid

**Võrk ja teenused:**

- [ ] ClusterIP Service DNS töötab
- [ ] NodePort Service kättesaadav väljast
- [ ] PostgreSQL + Secret + Service töötavad koos

---

## Veaotsing

**GitHub Actions ei käivitu:** Kontrollige et workflow fail on täpselt `.github/workflows/` kaustas ja branch on `main`.

**Docker push ebaõnnestub:** Kontrollige repo Settings → Actions → Workflow permissions → Read and write.

**kubectl ei tööta:** Proovige `sudo kubectl get nodes`. Kui see töötab, siis kubeconfig pole kopeeritud. Jooksutage uuesti kubeconfig seadistuse käsud.

**Pod on CrashLoopBackOff:** Käivitage `kubectl logs <pod-nimi>` et näha mis viga on. Kui logid on tühjad, käivitage `kubectl describe pod <pod-nimi>` ja vaadake Events sektsiooni.

**Service ei vasta:** Käivitage `kubectl get endpoints <service-nimi>`. Kui endpoint'ide list on tühi, siis Service'i selector ei klapi Pod'ide label'itega. Võrrelge selector ja labels väärtusi.

**ImagePullBackOff:** Image nimi või tag on vale. Kontrollige `image:` välja kirjavigu.

| Sümptom | Põhjus | Lahendus |
|---------|--------|----------|
| `ImagePullBackOff` | Vale image nimi/tag | Kontrolli kirjaviga |
| `CrashLoopBackOff` | Konteiner crashib | `kubectl logs` - vaata viga |
| `Pending` | Pole ressursse | `kubectl describe node` |
| Service tühi endpoints | Selector ei klapi | Võrdle labels ja selector |

---

## Refleksioon

Kirjuta paar lauset iga küsimuse kohta.

Mis oli kõige raskem osa selles laboris? Kas YAML süntaks, pipeline debug, või Kubernetes kontseptsioonid?

Milline oli sinu suurim "ahaa!" hetk? Mis hetkel said aru kuidas CI/CD või Kubernetes tegelikult töötab?

Kuidas sa selgitaksid sõbrale mis on CI/CD ja Kubernetes?

Kuidas need tööriistad aitaksid sind tulevases töökohas?

Mis oli kõige huvitavam osa?

---

## Dokumentatsioon

- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [k3s Documentation](https://docs.k3s.io/)
- [Kubernetes Docs](https://kubernetes.io/docs/home/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
