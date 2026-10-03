# Часть A — архитектура (M01)

Формат ответа: команда + что увидел + вывод.

## A1. Узлы, рантайм, control-plane
- команда: `kubectl get nodes -o wide`
- ответ: 2 узла — `cka` (ROLES=control-plane) и `cka-m02` (ROLES=`<none>`, воркер).
  CONTAINER-RUNTIME на обоих — `containerd://2.3.4`. Control-plane узел — `cka`.

## A2. Четыре компонента control plane и почему суффикс — имя узла
- команда: `kubectl get pods -n kube-system -o wide`
- ответ: `etcd-cka`, `kube-apiserver-cka`, `kube-controller-manager-cka`, `kube-scheduler-cka`.
  Суффикс `-cka` — это имя **узла**, а не контекста. Это static pods: kubelet поднимает их
  из файлов-манифестов на диске и регистрирует в API как mirror pods с именем
  `<имя манифеста>-<имя узла>`. Их не создаёт Deployment/ReplicaSet, поэтому в имени нет хешей.

## A3. Каталог static pods на узле + листинг
- команда: `minikube -p cka ssh -- ls -l /etc/kubernetes/manifests/`
- ответ: каталог `/etc/kubernetes/manifests` (задаётся `staticPodPath` в конфиге kubelet,
  `/var/lib/kubelet/config.yaml`). Листинг:
  ```
  etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
  ```
  kubelet читает этот каталог напрямую с диска, поэтому поднимает control plane даже когда
  API server лежит.

## A4. Сколько kube-proxy и почему; кто ими управляет
- команда: `kubectl get ds -n kube-system kube-proxy`, `kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide`
- ответ: 2 пода (`kube-proxy-fkggj` на `cka`, `kube-proxy-8gh24` на `cka-m02`) — по одному на
  каждый узел. Управляются **DaemonSet `kube-proxy`**: DaemonSet гарантирует ровно один под на
  каждом подходящем узле. Нет kube-proxy на узле → на этом узле не работают Services.

## A5. DNS: Service, namespace, ClusterIP, Deployment, реплики
- команда: `kubectl -n kube-system get svc kube-dns`, `kubectl -n kube-system get deploy coredns`
- ответ: Service `kube-dns` (историческое имя) в namespace `kube-system`, ClusterIP `10.96.0.10`.
  За ним Deployment `coredns`, **1 реплика** (READY 1/1; под `coredns-7d764666f9-tqbjs`,
  метка `k8s-app=kube-dns`).

## A6. Service kubernetes в default: ClusterIP и зачем
- команда: `kubectl get svc kubernetes -n default`
- ответ: ClusterIP `10.96.0.1`, порт 443. Это стабильный адрес **самого API server** внутри
  кластера: любой под ходит к API по `kubernetes.default.svc.cluster.local` (так работают
  контроллеры, операторы, CoreDNS и т.д.), не зная реального IP control-plane узла.

## A7. cluster-info упал — кластер или kubeconfig? почему
- команда: `kubectl cluster-info`
- ответ: в первую очередь подозреваю **kubeconfig**. `cluster-info` просто идёт по адресу
  из kubeconfig, и чаще всего проблема там: не тот контекст, протухший сертификат, неверный
  адрес/порт сервера. Порядок проверки: `kubectl config current-context` →
  `kubectl cluster-info` → только потом лезть на узлы (kubelet, static pods).

## A8. Short names (10 ресурсов) + команда, которой их получить
- команда: `kubectl api-resources`
- ответ:
  | ресурс | short name |
  |---|---|
  | pods | `po` |
  | services | `svc` |
  | deployments | `deploy` |
  | configmaps | `cm` |
  | namespaces | `ns` |
  | serviceaccounts | `sa` |
  | persistentvolumeclaims | `pvc` |
  | ingresses | `ing` |
  | networkpolicies | `netpol` |
  | endpointslices | **нет короткого имени** (пишется полностью) |

## A9. CNI / CRI / CSI + какой CNI-плагин в этом кластере
- команда: `kubectl get ds -n kube-system`, `kubectl get pods -n kube-system -l app=kindnet -o wide`
- ответ:
  - **CRI** (Container Runtime Interface) — как kubelet общается с рантаймом контейнеров (здесь containerd).
  - **CNI** (Container Network Interface) — выдача IP подам и сеть между подами/узлами.
  - **CSI** (Container Storage Interface) — подключение томов из внешних систем хранения.

  CNI-плагин в кластере — **kindnet**: DaemonSet `kindnet`, поды `kindnet-pmvpq` (на `cka`)
  и `kindnet-6md5q` (на `cka-m02`), по одному на узел, все Running. Признак сломанного CNI —
  узлы `NotReady` и поды, застрявшие в `ContainerCreating`.
