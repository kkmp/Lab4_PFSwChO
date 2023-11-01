# Laboratorium 4 - sprawozdanie

Zadanie rozpocząłem od wydania polecenia, które pozwoliło mi utworzyć przestrzeń nazw *lab4*:
```
kubectl create ns lab4
```

Następnie wydałem komendę, która pozwoliła mi na utworzenie pliku *my-quota.yaml*, zawierającego definicję obiektu quota o nazwie *my-quota*. Obiekt ten jest odpowiedzialny za zarządzanie zasobami w przestrzeni nazw *lab4*. Użyłem opcji *--hard*, która zgodnie z dokumentacją, ustawia twardy limit zasobów na 1 GiB pamięci RAM, 1000 milicore CPU oraz możliwość uruchomienia 5 podów:
```
kubectl create quota my-quota -n lab4 -o yaml --hard=cpu=1000m,memory=1Gi,pods=5 --dry-run=client > my-quota.yaml
```

Plik został dołączony do repozytorium. Umożliwia on kontrolę poprawności wydanej komendy. Wszystkie parametry zostały zdefiniowane zgodnie z wymaganiami (parametr dla cpu 1000m został zgodnie z logiką zapisany jako wartość 1).
Warto zauważyć, że możliwe jest utworzenie alternatywnej wersji pliku yaml, w którym argumenty *cpu* i *memory* opatrzone byłyby dodatkowaymi kluczami "limits" (*limits.cpu*, *limits.memory*), wówczas możliwe jest jednoczesne wskazanie minimalnych zasobów dla kontenerów (analogicznie klucz "requests"). Sprawozdanie dotyczy tylko limitów, dlatego też zdecydowałem się na pozostawienie pliku bez modyfikacji.

Kolejnym krokiem było utworzenie obiektu quota zgodnie z wygenerowanym plikiem yaml:
```
kubectl apply -f my-quota.yaml 
```

Poprawność wykonanych czynności sprawdziłem na dwa sposoby:
```
kubectl get quota -n lab4
NAME       AGE   REQUEST                              LIMIT
my-quota   60s   cpu: 0/1, memory: 0/1Gi, pods: 0/5   

kubectl describe quota my-quota -n lab4
Name:       my-quota
Namespace:  lab4
Resource    Used  Hard
--------    ----  ----
cpu         0     1
memory      0     1Gi
pods        0     5
```

Jak łatwo zauważyć wszystkie parametry limitów zostały ustawione zgodnie z oczekiwaniami. Żadne pody nie zostały jeszcze utworzone, zatem zasoby nie są też wykorzystywane.

W kolejnym kroku utworzyłem Deployment o nazwie *restrictednginx* w przestrzeni nazw *lab4* z trzema podami:
```
kubectl create deploy restrictednginx --image=nginx -n lab4 --replicas=3 
```

Następnie, zgodnie z treścią zadania, zdefiniowałem minimalne zasoby (opcja *--requests*) oraz górny limit zasobów (opcja *--limits*). Konfigurację wyeksportowałem do pliku *restrictednginx-resources.yaml* i dołączyłem do repozytorium:
```
kubectl set resources deployment restrictednginx --requests=cpu=125m,memory=64Mi --limits=cpu=250m,memory=256Mi -n lab4 -o yaml --dry-run=client > restrictednginx-resources.yaml
```

Analiza sekcji *resources* pozwoliła mi potwierdzić poprawność konfiguracji. W dalszej kolejności wydałem komendę, która umożliwiła mi wprowadzenie zmian zgodnie z utworzonym plikiem:
```
kubectl apply -f restrictednginx-resources.yaml
```

Poprawność uruchomienia trzech podów sprawdziłem za pomocą następujących poleceń:
```
kubectl get deploy -n lab4
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
restrictednginx   3/3     3            3           2m11s

kubectl get pods -n lab4
NAME                               READY   STATUS    RESTARTS   AGE
restrictednginx-5dcd9dc88c-9gslw   1/1     Running   0          38s
restrictednginx-5dcd9dc88c-hdtdm   1/1     Running   0          31s
restrictednginx-5dcd9dc88c-kvdrh   1/1     Running   0          34s
```

Wszystkie pody zostały uruchomione, jednak w celu uzyskania pewności, że nie wystąpiły żadne problemy, wydałem komendy, których sekcje poświęcone wydarzeniom ("Events") potwierdziły pomyślne uruchomienie Deploymentu, a także poszczególnych podów.

Deployment:
```
kubectl describe deploy restrictednginx -n lab4
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  7m49s  deployment-controller  Scaled up replica set restrictednginx-5cd867d5bb to 3
  Normal  ScalingReplicaSet  6m40s  deployment-controller  Scaled up replica set restrictednginx-5dcd9dc88c to 1
  Normal  ScalingReplicaSet  6m36s  deployment-controller  Scaled down replica set restrictednginx-5cd867d5bb to 2 from 3
  Normal  ScalingReplicaSet  6m36s  deployment-controller  Scaled up replica set restrictednginx-5dcd9dc88c to 2 from 1
  Normal  ScalingReplicaSet  6m33s  deployment-controller  Scaled down replica set restrictednginx-5cd867d5bb to 1 from 2
  Normal  ScalingReplicaSet  6m33s  deployment-controller  Scaled up replica set restrictednginx-5dcd9dc88c to 3 from 2
  Normal  ScalingReplicaSet  6m30s  deployment-controller  Scaled down replica set restrictednginx-5cd867d5bb to 0 from 1
```

Jeden z podów:
```
kubectl describe pod restrictednginx -n lab4
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  7m28s  default-scheduler  Successfully assigned lab4/restrictednginx-5dcd9dc88c-kvdrh to minikube
  Normal  Pulling    7m28s  kubelet            Pulling image "nginx"
  Normal  Pulled     7m27s  kubelet            Successfully pulled image "nginx" in 1.478104422s (1.47811594s including waiting)
  Normal  Created    7m26s  kubelet            Created container nginx
  Normal  Started    7m26s  kubelet            Started container nginx
```

Wydarzenia dla wszystkich pozostałych podów są analogiczne do zaprezentowanego. Dodatkowo sprawdziłem zużycie zasobów dla quoty, które jest zgodne z wprowadzoną konfiguracją (CPU 3\*125m oraz RAM 3\*64MiB):
```
kubectl describe quota my-quota -n lab4
Name:       my-quota
Namespace:  lab4
Resource    Used   Hard
--------    ----   ----
cpu         375m   1
memory      192Mi  1Gi
pods        3      5

```

Cel zadania został osiągnięty.
