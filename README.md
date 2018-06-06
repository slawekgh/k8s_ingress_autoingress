# k8s_ingress_autoingress

W niniejszym artykule poruszę temat wystawiania mikroserwisów na zewnątrz klastrów z kontenerami, będzie też troche o tym czy i jak zautomatyzować proces zarządzania kubernetesowym ingressem i jak z grubsza ogarnąć temat takiej automatyzacji w środowisku RBAC k8s. 
Zrobimy też sobie automat do dynamicznej rekonfiguracji ingressa i nawet dla potomnych wrzucimy go na najnowszy nabytek microsoftu. 


Najpierw jednak tytułem wstępu i zbudowania jakichkolwiek podwalin teoretycznych prześledzimy ogólne zagadnienie wystawiania usług kontenerowych na zewnątrz. 

Zacznijmy może od tego co tam słychać w 12-factorapp na ten temat - a konkretnie tu:
```
https://12factor.net/pl/port-binding
```
a że 12factor ma dużo do powiedzenia na każdy temat to i tu jest nie inaczej - w rozdziale 7 pojawia się zjawisko Port bindingu, cytuję:


"Udostępniaj usługi przez przydzielanie portów[...]
W przypadku aplikacji wdrożonej w środowisku produkcyjnym zapytania do udostępnionej publicznie nazwy hosta są obsługiwane przez warstwę nawigacji. Kierowane są one później do procesu sieciowego udostępnionego na danym porcie."


O ile cenię 12factor za upraszczanie życia i mądre wytyczne to ten rozdział jakoś do mnie nie przemawia i widzę w nim pewne niebezpieczeństwo nadinterpretacji. Wydaje mi się że dotknięto tu tematu pojedyńczych mikroserwisów a nie całych usług. Spotkałem się kilka razy z interpretacją tego rozdziału która prowadziła wprost do wystawiania usług na portach i kombinowania potem jak przemapowwać nazwy na porty albo urle na porty. 

Tu warto od razu wspomnieć o metodzie wystawiania usług jaka została zaimplementowana na początku istnienia docker swarma czyli mapowanie całych usług na porty. I znowu kombinowanie z mapowaniem itd - na pewno poprawiono to później i dodano wszystkie inne metody co przyczynia się do stale rosnącej popularności swarma. A nie, czekajcie... 

Z kolei jak wspomnimy o dockerowym EXPOSE w Dockerfile i opcjach "-p" lub "-P" to jakoś nagle można zdać sobie sprawę że zagadnienie jest przesuwane z miejsca w miejsce i dopiero profesjonalne orkiestratory jakoś zaczęły się tym zajmować i to ogarniać (w miarę). 

Mesos/Marathon np radzi sobie z tym tak że owszem inwentaryzuje exposowane porty ale potem i tak trzeba dorzucać sobie zewnętrzny api gateway który robi vhost mapping lub uri mapping, można też korzystać z marathon load-balancera lub pakować wszystko do consula i czytać z consula. Słabe i masa dłubania, z drugiej strony tu jest lepsze load-balancowanie na L7 niż jak w swarm na IPVS. 

A Kubernetes jak to z nim zwykle bywa dostarcza nam kilka metod a jak nie dostarcza takiej funkcjonalności jaką chcemy to można korzystać z zasady "battery included but removable" - obecnie community k8s oferuje liczne warianty.

Co do dostepnych opcji w środowisku k8s on premise zakładam że wszyscy wiedzą co i jak - ale jak nie wiedzą to service może być typu ClusterIp lub NodePort. Typ ClusterIp jest defaultowy i ogranicza wystawienie VIPa w środku klastra wiec wiele nam nie pomoże, z kolei NodePort jest z grubsza tożsamy ze swarmowym expose service czyli każdy węzeł wystawia port dla usługi (nawet jak nie ma jej kontenerów u siebie) i wewnętrznym IPVSem i iptablesem kieruje ruch i od razu robi load-balancing. Porty trzeba jakoś inwentaryzować, można nakłonić developerów żeby sobie z tym radzili, ja jednak zwykle spotykałem się z panicznym oporem z ich strony, co więcej równie często spotykałem się z oporem przed uri-mapping i parcie w stronę vhost mapping ale to już inna historia, 

Tu warto dodać że istnieją pewne opracowania sugerujące dostęp do usług via kubernetes-proxy ale wyłącznie na chwilę i dla środowisk domowych/developerskich - o tym jak poważny problem można sobie wygenerować używając na stałe na produkcji rozwiązań przeznaczonych do domowego labu przekonała się niedawno Tesla której infrastruktura kubernetesa zamiast pracować dla Tesli zaczęła kopać bitcoiny jakiemuś nieznanemu podmiotowi :-) 

```
https://blog.heptio.com/on-securing-the-kubernetes-dashboard-16b09b1b7aca
```

Reasumując:
- defaultowy typ k8s-service (ClusterIp) jest nieprzydatny (nie realizuje dostępu do usługi z zewnątrz)
- drugi typ (NodePort) wystawia usługi na port wzorem swarma więc jego przydatność jest dyskusyjna. 
- nie powinno sie używać k8s proxy chyba że na chwilę albo w domowym labie
- W chmurach można korzystać jeszcze z typu LoadBalancer który wysteruje chmurowym LB ale tu pojawiają się koszty - każdy serwice będzie chciał powoływać swój LB i po chwili namnożą sie zewnętrzne IP i kwota na fakturze znacząco wzrośnie

Aby zaadresować powyższe problemy w k8s wymyślono Ingress i niejako zdelegowano na community proces wytwórczy ingress controllera 

Czym jest Ingress? Biorąc pod uwagę że z zasady rozwiązuje wszystkie wcześniej omawiane problemy można śmiało samemu zdefiniować jego architekturę:

1. skoro wystawianie na porty jest passe to Ingress pewnie umie wystawiać servisy na URL'e lub na vHosty 
2. skoro w chmurach dużo płacimy za zewnętrzny IP dla chmurowego LoadBalancera to Ingress pewnie umie wystawić wszystkie usługi k8s w jednym miejscu 
3. Ingress pewnie dobrze żeby był jednym z kubernetes resource i można nim było sterować via kubectl/yaml/helm czy co tam jeszcze chcemy 
4. Fajnie jakby Ingress jakoś sam automatycznie dodawał nowe serwisy jeśli zostały odpowiednio olabelowane 
5. Ingress powinien mieć modularną budowę - tak aby dało się zmieniać jego silnik w zależności od tego czy ktoś lubi traefika czy nginxa czy inne 
6. Powiniem umieć routować do wszystkich usług k8s na jednym IP (z tym rutowaniem to może przesadziłem, bardziej reverse-proxy tu pasuje)


Jak łatwo się domyślić prawie wszystkie powyższe punkty są spełnione i Ingress z powodzeniem realizuje te założenia. Poza jednym punktem który jest mocno dyskusyjny - czyli 4. 

No wlasnie - co z tym punktem o automatyce? Osobiście tak do końca nie mogę rekomendować czy lepiej dać swobodę developerom czy jednak manualnie kontrolować co nasz klaster wystawia na zewnątrz. Bardzo dużo zależy od środowiska, ludzi, standaryzacji, strefy wpływu bezpieczników w firmie itd itp. 
Różnie bywało i w niektórych projektach stosowałem ręczny proces kontroli, w innych robiłem automaty które wystawiały usługi na zewnątrz bez udziału adminów. Zawsze gdzieś tam górę brało podejście oparte na pełnej automatyce i co najwyżej nadzorowania/auditingu, przy 10 deploymentach na godzinę średnio chciałoby mi sie modyfikować jakieś wpisy czy rekonfigurować cokolwiek. 
To ogólnie temat na głębszą dyskusję ale fakt faktem zawsze lepiej mieć automat niż go nie mieć, najwyżej nie będzie uruchomiony - dlatego w dalszej części go sobie zbudujemy i wspawamy w infrastrukturę kubernetesa. 

Wracając do ingressa - twór ten podzielony jest na 2 komponenty 
- Ingress resource (czyli kolejny k8s resource obok Deployments, Services , ReplicaSets itd)
- Ingress controller (nginx, traefik, istio itd) 

Z racji długoletniego używania nginx do realizacji load-balancingu dla kontenerów zdecydowałem się na zastosowanie jako controllera właśnie tego silnika. Słyszałem też dużo dobrych opinii o traefiku ale niestety równie dużo wzmianek że nie nadaje się jeszcze na produkcję - jednakże nie testowałem, nie wiem, nie znam, mam go na roadmapie, jak coś będe wiedział dam znać. 

Wracając do nginxa - realizowany w oparciu o niego Ingress Controller oparty jest o dwa komponenty:
- nginx-rev-proxy
- backend-service (do rzucania 404 gdy nie znajdzie się dany service)

Repo projektu a właściwie link do procedury instalacji znajduje się tutaj:
```
https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md
```
Instalacja polega w pierwszym kroku na wgraniu do kubernetesa Yamla z sekcji Mandatory Commands:
```
https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

czyli po kolei co tam w środku jest robione:
- powołanie namespace ingress-nginx
- Deployment dla default-http-backend (tego od rzucania 404 w stronę niestosownych zapytań) 
- Service dla default-http-backend
- 3 ConfigMapy: nginx-configuration, tcp-services, udp-services
- ServiceAccount nginx-ingress-serviceaccount
- ClusterRole nginx-ingress-clusterrole
- Role nginx-ingress-role
- Deployment nginx-ingress-controller 

w drugim kroku trzeba wgrać yamla dla opcji bare-metal (czyli on-premise):
```
https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```
tutaj z kolei mamy utworzenie Service typu NodePort dla ingress-nginx (nasz Ingress controller będzie dostepny na każdym węźle na konkretnym porcie)

Wykonanie powyższych kroków powoduje że na klastrze k8s mamy Ingress Controller - pozostaje jeszcze dorzeźbić sam IngressResource.

Przy okazji - nasz ingress controller słucha na każdym węźle na porcie 32165:
```
# kubectl  get svc  -n ingress-nginx
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
default-http-backend   ClusterIP   10.106.132.195   <none>        80/TCP                       8m
ingress-nginx          NodePort    10.109.196.242   <none>        80:32165/TCP,443:31729/TCP   7m
```

Aby jednak mieć cokolwiek do wystawiania i testów trzeba powołać 2 serwisy (i leżące pod nimi 2 deploymenty) 
```
kubectl run website --image=gimboo/apacz --port=80
kubectl run forums --image=gimboo/apacz --port=80
kubectl expose deployment/website
kubectl expose deployment/forums
```

obraz gimboo/apacz to testowy obraz występujący dosyć często w tej serii blogpostów :-) 
jego jedyną funkcjonalnością jest wystawianie HOSTNAME kontenera na porcie 80 - do testów wystarczy jak znalazł 

Pozostaje teraz stworzyć IngressResource

Zaczynamy od najprostrzego ingressa bez żadnych reguł (taki ingress wrzuca wszystko jak leci na 1 serwis):
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  backend:
    serviceName: website
    servicePort: 80
```
robimy kubectl apply -f powyższy_plik  i testujemy konfig 

```
# curl 127.0.0.1:32165 
website-7dcbbddc8f-cx4dc
```

skalujemy backend (deploy pod spodem) do 2 i sprawdzamy czy działa Load-Balancing

```
# kubectl scale deploy website --replicas=2
deployment.extensions "website" scaled
# curl cent401:32165
website-7dcbbddc8f-cx4dc
# curl cent401:32165
website-7dcbbddc8f-qd2np
```

jest ok , jak widać działa, czas na ingressa z rozdziałem ruchu na dwa serwisy pod spodem:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: www.mysite.com
    http:
      paths:
      - backend:
          serviceName: website
          servicePort: 80
  - host: forums.mysite.com
    http:
      paths:
      - path:
        backend:
          serviceName: forums
          servicePort: 80
```

czyli jak wpadnie na hosta www.mysite.com to przerzuci na service=website a jak na forums.mysite.com to na service=forums 

po zaaplikowaniu powyższego tak oto wygląda działanie: 
```
# curl -H 'Host:www.mysite.com'  127.0.0.1:32165
website-7dcbbddc8f-cx4dc

# curl -H 'Host:forums.mysite.com'  127.0.0.1:32165
forums-6d4b76fd95-vtrcw

# curl -H 'Host:fake.mysite.com'  127.0.0.1:32165
default backend - 404
```

osiągneliśmy zatem vHost mapping , czas na url mapping 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: www.mysite.com
    http:
      paths:
      - path: /website
        backend:
          serviceName: website
          servicePort: 80
      - path: /forums
        backend:
          serviceName: forums
          servicePort: 80
```

tym razem mechanizm polega na wrzucaniu na dany serwis ruchu na podstawie url w zapytaniu 
po zaaplikowaniu działa to następująco:

```
# curl -H 'Host:www.mysite.com' 127.0.0.1:32165/forums
forums-6d4b76fd95-vtrcw
# curl -H 'Host:www.mysite.com' 127.0.0.1:32165/website
website-7dcbbddc8f-cx4dc
# curl -H 'Host:www.mysite.com' 127.0.0.1:32165/website
website-7dcbbddc8f-qd2np
# curl -H 'Host:www.mysite.com' 127.0.0.1:32165/fakeurl
default backend - 404
```

Ogólnie jak widać działa i daje nawet jakieś niezerowe możliwości konfiguracji - jeden jedyny problem jaki mnie uwiera to ta nieszczęsna konieczność ręcznego dostawiania wpisów do ingressa dla nowo powołanych serwisów - sprawdzałem pobieżnie czy nie ma jakiegoś gotowego rozwiązania i w sumie znalazłem jeden projekt ale nie daje on oznak życia (w całym repo zarejestrowane jedno (!) issue, zresztą przeze mnie i na chwilę pisania niniejszego tekstu bez jakiejkolwiek odpowiedzi) . 
```
https://github.com/hxquangnhat/kubernetes-auto-ingress
```

Z drugiej strony jak sie dobrze zastanowić to zrobienie automatu nie będzie zbyt trudne i większość walki będzie z k8s RBAC niż z samym mechanizmem automatycznego update. A przy okazji możemy się czegoś nowego nauczyć :-) 

A zatem czas stworzyć nasz własny testowy autoingress, oto założenia (przypominamy że od 1.10 mamy default=RBAC):

1. nowo wystawiane serwisy będą miały olabelowanie wskazujące autoingressowi jak ma parsować i dodawać do ingressa urle
będzie to zrobione na labelce "auto_ingress" - dla uproszczenia na potrzeby tego LABu pakuję wszystkie 3 kluczowe informacje w jeden string 

```
metadata:
  labels:
    run: serwis5
    auto_ingress: 'serwis5_path_80'
```

2. autoingress będzie oglądał sobie wszystkie serwisy na k8s i dla każdego czytał dane z tego labelu i parsował to do 3 wartości 
service-name, service-url-path, service-port 
na tej podstawie będzie sterował ingressem który z kolei będzie działał w namespace=default 

3. autoingress będzie działał we własnym namespace="autoingress" - tak żeby nie przeszkadzał innym i w drugą stronę 

4. powołamy będziie dla niego ServiceAccount=autoingress-serviceaccount
w przypadku naszego automatu będziemy poniższym userem: 
```
User "system:serviceaccount:autoingress:autoingress-serviceaccount" 
```

5. powołana zostanie ClusterRole=autoingress-clusterrole która będzie miała następujące uprawnienia:
```
- apiGroups:
      - ""
    resources:
      - services
    verbs:
      - list
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
      - patch
      - delete
      - create
```

te pierwsze uprawnienia to czytanie info o serwisach - będę to robił via:
```
kubectl -n default get svc -o jsonpath='{.items[*].metadata.labels.auto_ingress}')
```
te drugie uprawnienia to update ingressa , nie za bardzo chciało mi się wnikać więc dałem dosyć szeroko 

5. ServiceAccount autoingressa będzie zbindowany z rolą autoingress-clusterrole

6. Powołana będzie ConfigMapa=autoingress-configuration z której autoingress będzie czytał konfig - a zasadniczo 3 dane (tu moje deafulty):
```
  INGRESSHOST: www.mysite.com
  INGRESSNAME: my-ingress
  INGRESSNAMESPACE: default
```

7. Finalnie deployment=autoingress korzystający z gotowego obrazu "demona" gimboo/autoingress:1.0
(z tym demonem to bym może nie przesadzał, napisałem to na potrzeby tego LABU w shellu)

to teraz te punkty 1-7 trzeba wpakować do yamla

finalnie jak się komuś nie chce nad tym wszystkim zastanawiać to można na skróty jednym poleceniem wykonać deploy na czystym klastrze k8s:

```
# kubectl apply -f https://raw.githubusercontent.com/slawekgh/autoingress/master/autoingress.yaml

namespace "autoingress" created
serviceaccount "autoingress-serviceaccount" created
clusterrole.rbac.authorization.k8s.io "autoingress-clusterrole" created
clusterrolebinding.rbac.authorization.k8s.io "autoingress-clusterrole-binding" created
configmap "autoingress-configuration" created
deployment.extensions "autoingress" created
```

to teraz pozostaje kreować serwisy i cieszyć się że Ingress sam sie rekonfiguruje 

```
# cat service1.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    run: serwis1
    auto_ingress: 'serwis1_serwis1_80'
  name: serwis1
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: website

# kubectl apply -f service1.yaml 
service "serwis1" created

# kubectl  get ing -o yaml
[...]
  spec:
    rules:
    - host: www.mysite.com
      http:
        paths:
        - backend:
            serviceName: serwis1
            servicePort: 80
          path: /serwis1


# curl -H 'Host:www.mysite.com' 127.0.0.1:32165/serwis1
website-7dcbbddc8f-qd2np

# curl -H 'Host:www.mysite.com' 127.0.0.1:32165/serwis2
default backend - 404

```
jak widać tego drugiego nie ma - trzeba go zrobić zatem

```
# cat service2.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    run: serwis2
    auto_ingress: 'serwis2_serwis2_80'
  name: serwis2
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: website

# kubectl apply -f  service2.yaml 
service "serwis2" created
```

Ingress już się zmienił :

```
# kubectl  get ing -o yaml
  spec:
    rules:
    - host: www.mysite.com
      http:
        paths:
        - backend:
            serviceName: serwis1
            servicePort: 80
          path: /serwis1
        - backend:
            serviceName: serwis2
            servicePort: 80
          path: /serwis2
          
# curl -H 'Host:www.mysite.com' 127.0.0.1:32165/serwis2
website-7dcbbddc8f-qd2np
```

Od tej pory jeśli chcemy żeby nasze serwisy wystawiały się automatycznie na ingresie trzeba im dodać label auto_ingress i gotowe. Wszystko dzieje się automatycznie i samo za nas :-) 


![Alt Text](https://media1.tenor.com/images/5772c0dd36dd7440c449afc8dd50d913/tenor.gif?itemid=4791402)


To by było na tyle, pozostaje jeszcze dyskusja o komunikacji inter-kontenerowej , czy narzucić obowiązek rozmawiania zawsze przez ingressa czy zezwolić na odwoływanie sie po nazwach serwisów wewnątrz namespace - na chwilę obecną nie mam gotowej recepty. 

Pierwszy wariant wnosi troche porządku i standaryzacji ale z drugiej strony rezygnujemy wtedy z funkcjonalności wbudowanych w k8s , z przyjemnością poznałbym opinie innych użytkowników kubernetesa. 


