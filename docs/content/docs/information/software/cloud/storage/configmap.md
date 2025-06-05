---
weight: 2
title: "ConfigMap"
draft: false
---

其实 ConfigMap 功能在 Kubernetes1.2 版本的时候就有了，许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。这些配置信息需要与 docker image 解耦，你总不能每修改一个配置就重做一个 image 吧？ConfigMap API 给我们提供了向容器中注入配置信息的机制，ConfigMap 可以被用来保存单个属性，也可以用来保存整个配置文件或者 JSON 二进制大对象。

## ConfigMap 概览

**ConfigMap API** 资源用来保存 **key-value pair** 配置数据，这个数据可以在 **pods** 里使用，或者被用来为像 **controller** 一样的系统组件存储配置数据。虽然 ConfigMap 跟 [Secrets](https://kubernetes.io/docs/user-guide/secrets/) 类似，但是 ConfigMap 更方便的处理不含敏感信息的字符串。 注意：ConfigMaps 不是属性配置文件的替代品。ConfigMaps 只是作为多个 properties 文件的引用。你可以把它理解为 Linux 系统中的 `/etc` 目录，专门用来存储配置文件的目录。下面举个例子，使用 ConfigMap 配置来创建 Kubernetes Volumes，ConfigMap 中的每个 data 项都会成为一个新文件。

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: example-config
  namespace: default
data:
  example.property.1: hello
  example.property.2: world
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

`data` 一栏包括了配置数据，ConfigMap 可以被用来保存单个属性，也可以用来保存一个配置文件。 配置数据可以通过很多种方式在 Pods 里被使用。ConfigMaps 可以被用来：

1. 设置环境变量的值
2. 在容器里设置命令行参数
3. 在数据卷里面创建 config 文件

用户和系统组件两者都可以在 ConfigMap 里面存储配置数据。

其实不用看下面的文章，直接从 `kubectl create configmap -h` 的帮助信息中就可以对 ConfigMap 究竟如何创建略知一二了。

```
Examples:
  # Create a new configmap named my-config based on folder bar
  kubectl create configmap my-config --from-file=path/to/bar

  # Create a new configmap named my-config with specified keys instead of file basenames on disk
  kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt

  # Create a new configmap named my-config with key1=config1 and key2=config2
  kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2
```

## 创建 ConfigMaps

可以使用该命令，用给定值、文件或目录来创建 ConfigMap。

```
kubectl create configmap
```

### 使用目录创建

比如我们已经有了一些配置文件，其中包含了我们想要设置的 ConfigMap 的值：

```bash
$ ls docs/user-guide/configmap/kubectl/
game.properties
ui.properties

$ cat docs/user-guide/configmap/kubectl/game.properties
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30

$ cat docs/user-guide/configmap/kubectl/ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
```

使用下面的命令可以创建一个包含目录中所有文件的 ConfigMap。

```bash
$ kubectl create configmap game-config --from-file=docs/user-guide/configmap/kubectl
```

`—from-file` 指定在目录下的所有文件都会被用在 ConfigMap 里面创建一个键值对，键的名字就是文件名，值就是文件的内容。

让我们来看一下这个命令创建的 ConfigMap：

```yaml
$ kubectl describe configmaps game-config
Name:           game-config
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        158 bytes
ui.properties:          83 bytes
```

我们可以看到那两个 key 是从 kubectl 指定的目录中的文件名。这些 key 的内容可能会很大，所以在 kubectl describe 的输出中，只能够看到键的名字和他们的大小。 如果想要看到键的值的话，可以使用 `kubectl get`：

```bash
$ kubectl get configmaps game-config -o yaml
```

我们以 `yaml` 格式输出配置。

```yaml
apiVersion: v1
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:34:05Z
  name: game-config
  namespace: default
  resourceVersion: "407"
  selfLink: /api/v1/namespaces/default/configmaps/game-config
  uid: 30944725-d66e-11e5-8cd0-68f728db1985
```

### 使用文件创建

刚才**使用目录创建**的时候我们 `—from-file` 指定的是一个目录，只要指定为一个文件就可以从单个文件中创建 ConfigMap。

```bash
$ kubectl create configmap game-config-2 --from-file=docs/user-guide/configmap/kubectl/game.properties 

$ kubectl get configmaps game-config-2 -o yaml
apiVersion: v1
data:
  game-special-key: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:54:22Z
  name: game-config-3
  namespace: default
  resourceVersion: "530"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-3
  uid: 05f8da22-d671-11e5-8cd0-68f728db1985
```

`—from-file` 这个参数可以使用多次，你可以使用两次分别指定上个实例中的那两个配置文件，效果就跟指定整个目录是一样的。

### 使用字面值创建

使用文字值创建，利用 `—from-literal` 参数传递配置信息，该参数可以使用多次，格式如下；

```bash
$ kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm

$ kubectl get configmaps special-config -o yaml
apiVersion: v1
data:
  special.how: very
  special.type: charm
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: special-config
  namespace: default
  resourceVersion: "651"
  selfLink: /api/v1/namespaces/default/configmaps/special-config
  uid: dadce046-d673-11e5-8cd0-68f728db1985
```

## Pod 中使用 ConfigMap

**使用 ConfigMap 来替代环境变量**

ConfigMap 可以被用来填入环境变量。看下下面的 ConfigMap。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

我们可以在 Pod 中这样使用 ConfigMap：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
      envFrom:
        - configMapRef:
            name: env-config
  restartPolicy: Never
```

这个 Pod 运行后会输出如下几行：

```
SPECIAL_LEVEL_KEY=very
SPECIAL_TYPE_KEY=charm
log_level=INFO
```

**用 ConfigMap 设置命令行参数**

ConfigMap 也可以被使用来设置容器中的命令或者参数值。它使用的是 Kubernetes 的 $(VAR_NAME) 替换语法。我们看下下面这个 ConfigMap。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

为了将 ConfigMap 中的值注入到命令行的参数里面，我们还要像前面那个例子一样使用环境变量替换语法 `${VAR_NAME)`。（其实这个东西就是给 Docker 容器设置环境变量，以前我创建镜像的时候经常这么玩，通过 docker run 的时候指定 - e 参数修改镜像里的环境变量，然后 docker 的 CMD 命令再利用该 $(VAR_NAME) 通过 sed 来修改配置文件或者作为命令行启动参数。）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
  restartPolicy: Never
```

运行这个 Pod 后会输出：

```
very charm
```

**通过数据卷插件使用 ConfigMap**

ConfigMap 也可以在数据卷里面被使用。还是这个 ConfigMap。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

在数据卷里面使用这个 ConfigMap，有不同的选项。最基本的就是将文件填入数据卷，在这个文件中，键就是文件名，键值就是文件内容：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "cat /etc/config/special.how" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
  restartPolicy: Never
```

运行这个 Pod 的输出是 `very`。

我们也可以在 ConfigMap 值被映射的数据卷里控制路径。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh","-c","cat /etc/config/path/to/special-key" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: path/to/special-key
  restartPolicy: Never
```

运行这个 Pod 后的结果是 `very`。

## 热更新

ConfigMap是用来存储配置文件的kubernetes资源对象，所有的配置内容都存储在etcd中，下文主要是探究 ConfigMap 的创建和更新流程，以及对 ConfigMap 更新后容器内挂载的内容是否同步更新的测试。

### 测试示例

假设我们在 `default` namespace 下有一个名为 `nginx-config` 的 ConfigMap，可以使用 `kubectl`命令来获取：

```bash
$ kubectl get configmap nginx-config
NAME           DATA      AGE
nginx-config   1         99d
```

获取该ConfigMap的内容。

```bash
kubectl get configmap nginx-config -o yaml
```

```bash
apiVersion: v1
data:
  nginx.conf: |-
    worker_processes 1;

    events { worker_connections 1024; }

    http {
        sendfile on;

        server {
            listen 80;

            # a test endpoint that returns http 200s
            location / {
                proxy_pass http://httpstat.us/200;
                proxy_set_header  X-Real-IP  $remote_addr;
            }
        }

        server {

            listen 80;
            server_name api.hello.world;

            location / {
                proxy_pass http://l5d.default.svc.cluster.local;
                proxy_set_header Host $host;
                proxy_set_header Connection "";
                proxy_http_version 1.1;

                more_clear_input_headers 'l5d-ctx-*' 'l5d-dtab' 'l5d-sample';
            }
        }

        server {

            listen 80;
            server_name www.hello.world;

            location / {


                # allow 'employees' to perform dtab overrides
                if ($cookie_special_employee_cookie != "letmein") {
                  more_clear_input_headers 'l5d-ctx-*' 'l5d-dtab' 'l5d-sample';
                }

                # add a dtab override to get people to our beta, world-v2
                set $xheader "";

                if ($cookie_special_employee_cookie ~* "dogfood") {
                  set $xheader "/host/world => /srv/world-v2;";
                }

                proxy_set_header 'l5d-dtab' $xheader;


                proxy_pass http://l5d.default.svc.cluster.local;
                proxy_set_header Host $host;
                proxy_set_header Connection "";
                proxy_http_version 1.1;
            }
        }
    }
kind: ConfigMap
metadata:
  creationTimestamp: 2017-08-01T06:53:17Z
  name: nginx-config
  namespace: default
  resourceVersion: "14925806"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-config
  uid: 18d70527-7686-11e7-bfbd-8af1e3a7c5bd
```

ConfigMap中的内容是存储到etcd中的，然后查询etcd：

```bash
ETCDCTL_API=3 etcdctl get /registry/configmaps/default/nginx-config -w json|python -m json.tool
```

注意使用 v3 版本的 etcdctl API，下面是输出结果：

```json
{
    "count": 1,
    "header": {
        "cluster_id": 12091028579527406772,
        "member_id": 16557816780141026208,
        "raft_term": 36,
        "revision": 29258723
    },
    "kvs": [
        {
            "create_revision": 14925806,
            "key": "L3JlZ2lzdHJ5L2NvbmZpZ21hcHMvZGVmYXVsdC9uZ2lueC1jb25maWc=",
            "mod_revision": 14925806,
            "value": "azhzAAoPCgJ2MRIJQ29uZmlnTWFwEqQMClQKDG5naW54LWNvbmZpZxIAGgdkZWZhdWx0IgAqJDE4ZDcwNTI3LTc2ODYtMTFlNy1iZmJkLThhZjFlM2E3YzViZDIAOABCCwjdyoDMBRC5ss54egASywsKCm5naW54LmNvbmYSvAt3b3JrZXJfcHJvY2Vzc2VzIDE7CgpldmVudHMgeyB3b3JrZXJfY29ubmVjdGlvbnMgMTAyNDsgfQoKaHR0cCB7CiAgICBzZW5kZmlsZSBvbjsKCiAgICBzZXJ2ZXIgewogICAgICAgIGxpc3RlbiA4MDsKCiAgICAgICAgIyBhIHRlc3QgZW5kcG9pbnQgdGhhdCByZXR1cm5zIGh0dHAgMjAwcwogICAgICAgIGxvY2F0aW9uIC8gewogICAgICAgICAgICBwcm94eV9wYXNzIGh0dHA6Ly9odHRwc3RhdC51cy8yMDA7CiAgICAgICAgICAgIHByb3h5X3NldF9oZWFkZXIgIFgtUmVhbC1JUCAgJHJlbW90ZV9hZGRyOwogICAgICAgIH0KICAgIH0KCiAgICBzZXJ2ZXIgewoKICAgICAgICBsaXN0ZW4gODA7CiAgICAgICAgc2VydmVyX25hbWUgYXBpLmhlbGxvLndvcmxkOwoKICAgICAgICBsb2NhdGlvbiAvIHsKICAgICAgICAgICAgcHJveHlfcGFzcyBodHRwOi8vbDVkLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWw7CiAgICAgICAgICAgIHByb3h5X3NldF9oZWFkZXIgSG9zdCAkaG9zdDsKICAgICAgICAgICAgcHJveHlfc2V0X2hlYWRlciBDb25uZWN0aW9uICIiOwogICAgICAgICAgICBwcm94eV9odHRwX3ZlcnNpb24gMS4xOwoKICAgICAgICAgICAgbW9yZV9jbGVhcl9pbnB1dF9oZWFkZXJzICdsNWQtY3R4LSonICdsNWQtZHRhYicgJ2w1ZC1zYW1wbGUnOwogICAgICAgIH0KICAgIH0KCiAgICBzZXJ2ZXIgewoKICAgICAgICBsaXN0ZW4gODA7CiAgICAgICAgc2VydmVyX25hbWUgd3d3LmhlbGxvLndvcmxkOwoKICAgICAgICBsb2NhdGlvbiAvIHsKCgogICAgICAgICAgICAjIGFsbG93ICdlbXBsb3llZXMnIHRvIHBlcmZvcm0gZHRhYiBvdmVycmlkZXMKICAgICAgICAgICAgaWYgKCRjb29raWVfc3BlY2lhbF9lbXBsb3llZV9jb29raWUgIT0gImxldG1laW4iKSB7CiAgICAgICAgICAgICAgbW9yZV9jbGVhcl9pbnB1dF9oZWFkZXJzICdsNWQtY3R4LSonICdsNWQtZHRhYicgJ2w1ZC1zYW1wbGUnOwogICAgICAgICAgICB9CgogICAgICAgICAgICAjIGFkZCBhIGR0YWIgb3ZlcnJpZGUgdG8gZ2V0IHBlb3BsZSB0byBvdXIgYmV0YSwgd29ybGQtdjIKICAgICAgICAgICAgc2V0ICR4aGVhZGVyICIiOwoKICAgICAgICAgICAgaWYgKCRjb29raWVfc3BlY2lhbF9lbXBsb3llZV9jb29raWUgfiogImRvZ2Zvb2QiKSB7CiAgICAgICAgICAgICAgc2V0ICR4aGVhZGVyICIvaG9zdC93b3JsZCA9PiAvc3J2L3dvcmxkLXYyOyI7CiAgICAgICAgICAgIH0KCiAgICAgICAgICAgIHByb3h5X3NldF9oZWFkZXIgJ2w1ZC1kdGFiJyAkeGhlYWRlcjsKCgogICAgICAgICAgICBwcm94eV9wYXNzIGh0dHA6Ly9sNWQuZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbDsKICAgICAgICAgICAgcHJveHlfc2V0X2hlYWRlciBIb3N0ICRob3N0OwogICAgICAgICAgICBwcm94eV9zZXRfaGVhZGVyIENvbm5lY3Rpb24gIiI7CiAgICAgICAgICAgIHByb3h5X2h0dHBfdmVyc2lvbiAxLjE7CiAgICAgICAgfQogICAgfQp9GgAiAA==",
            "version": 1
        }
    ]
}
```

其中的value就是 `nginx.conf` 配置文件的内容。

可以使用base64解码查看具体值，关于etcdctl的使用请参考[使用etcdctl访问kuberentes数据](../guide/using-etcdctl-to-access-kubernetes-data.md)。

### 代码

ConfigMap 结构体的定义：

```go
// ConfigMap holds configuration data for pods to consume.
type ConfigMap struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: http://releases.k8s.io/HEAD/docs/devel/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Data contains the configuration data.
	// Each key must be a valid DNS_SUBDOMAIN with an optional leading dot.
	// +optional
	Data map[string]string `json:"data,omitempty" protobuf:"bytes,2,rep,name=data"`
}
```

在 `staging/src/k8s.io/client-go/kubernetes/typed/core/v1/configmap.go` 中ConfigMap 的接口定义：

```go
// ConfigMapInterface has methods to work with ConfigMap resources.
type ConfigMapInterface interface {
	Create(*v1.ConfigMap) (*v1.ConfigMap, error)
	Update(*v1.ConfigMap) (*v1.ConfigMap, error)
	Delete(name string, options *meta_v1.DeleteOptions) error
	DeleteCollection(options *meta_v1.DeleteOptions, listOptions meta_v1.ListOptions) error
	Get(name string, options meta_v1.GetOptions) (*v1.ConfigMap, error)
	List(opts meta_v1.ListOptions) (*v1.ConfigMapList, error)
	Watch(opts meta_v1.ListOptions) (watch.Interface, error)
	Patch(name string, pt types.PatchType, data []byte, subresources ...string) (result *v1.ConfigMap, err error)
	ConfigMapExpansion
}
```

在 `staging/src/k8s.io/client-go/kubernetes/typed/core/v1/configmap.go` 中创建 ConfigMap 的方法如下:

```go
// Create takes the representation of a configMap and creates it.  Returns the server's representation of the configMap, and an error, if there is any.
func (c *configMaps) Create(configMap *v1.ConfigMap) (result *v1.ConfigMap, err error) {
	result = &v1.ConfigMap{}
	err = c.client.Post().
		Namespace(c.ns).
		Resource("configmaps").
		Body(configMap).
		Do().
		Into(result)
	return
}
```

通过 RESTful 请求在 etcd 中存储 ConfigMap 的配置，该方法中设置了资源对象的 namespace 和 HTTP 请求中的 body，执行后将请求结果保存到 result 中返回给调用者。

**注意 Body 的结构**

```java
// Body makes the request use obj as the body. Optional.
// If obj is a string, try to read a file of that name.
// If obj is a []byte, send it directly.
// If obj is an io.Reader, use it directly.
// If obj is a runtime.Object, marshal it correctly, and set Content-Type header.
// If obj is a runtime.Object and nil, do nothing.
// Otherwise, set an error.
```

创建 ConfigMap RESTful 请求中的的 Body 中包含 `ObjectMeta` 和 `namespace`。

HTTP 请求中的结构体：

```go
// Request allows for building up a request to a server in a chained fashion.
// Any errors are stored until the end of your call, so you only have to
// check once.
type Request struct {
	// required
	client HTTPClient
	verb   string

	baseURL     *url.URL
	content     ContentConfig
	serializers Serializers

	// generic components accessible via method setters
	pathPrefix string
	subpath    string
	params     url.Values
	headers    http.Header

	// structural elements of the request that are part of the Kubernetes API conventions
	namespace    string
	namespaceSet bool
	resource     string
	resourceName string
	subresource  string
	timeout      time.Duration

	// output
	err  error
	body io.Reader

	// This is only used for per-request timeouts, deadlines, and cancellations.
	ctx context.Context

	backoffMgr BackoffManager
	throttle   flowcontrol.RateLimiter
}
```

### 测试

分别测试使用 ConfigMap 挂载 Env 和 Volume 的情况。

#### 更新使用ConfigMap挂载的Env

使用下面的配置创建 nginx 容器测试更新 ConfigMap 后容器内的环境变量是否也跟着更新。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: harbor-001.jimmysong.io/library/nginx:1.9
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: env-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

获取环境变量的值

```bash
$ kubectl exec `kubectl get pods -l run=my-nginx  -o=name|cut -d "/" -f2` env|grep log_level
log_level=INFO
```

修改 ConfigMap

```bash
$ kubectl edit configmap env-config
```

修改 `log_level` 的值为 `DEBUG`。

再次查看环境变量的值。

```bash
$ kubectl exec `kubectl get pods -l run=my-nginx  -o=name|cut -d "/" -f2` env|grep log_level
log_level=INFO
```

实践证明修改 ConfigMap 无法更新容器中已注入的环境变量信息。

#### 更新使用ConfigMap挂载的Volume

使用下面的配置创建 nginx 容器测试更新 ConfigMap 后容器内挂载的文件是否也跟着更新。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: harbor-001.jimmysong.io/library/nginx:1.9
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: special-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  log_level: INFO
```

```bash
$ kubectl exec `kubectl get pods -l run=my-nginx  -o=name|cut -d "/" -f2` cat /etc/config/log_level
INFO
```

修改 ConfigMap

```bash
$ kubectl edit configmap special-config
```

修改 `log_level` 的值为 `DEBUG`。

等待大概10秒钟时间，再次查看环境变量的值。

```bash
$ kubectl exec `kubectl get pods -l run=my-nginx  -o=name|cut -d "/" -f2` cat /etc/config/log_level
DEBUG
```

我们可以看到使用 ConfigMap 方式挂载的 Volume 的文件中的内容已经变成了 `DEBUG`。

Known Issue：
如果使用ConfigMap的**subPath**挂载为Container的Volume，Kubernetes不会做自动热更新:
https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#mounted-configmaps-are-updated-automatically

### ConfigMap 更新后滚动更新 Pod

更新 ConfigMap 目前并不会触发相关 Pod 的滚动更新，可以通过修改 pod annotations 的方式强制触发滚动更新。

```bash
$ kubectl patch deployment my-nginx --patch '{"spec": {"template": {"metadata": {"annotations": {"version/config": "20180411" }}}}}'
```

这个例子里我们在 `.spec.template.metadata.annotations` 中添加 `version/config`，每次通过修改 `version/config` 来触发滚动更新。

### 总结

更新 ConfigMap 后：

- 使用该 ConfigMap 挂载的 Env **不会**同步更新
- 使用该 ConfigMap 挂载的 Volume 中的数据需要一段时间（实测大概10秒）才能同步更新

ENV 是在容器启动的时候注入的，启动之后 kubernetes 就不会再改变环境变量的值，且同一个 namespace 中的 pod 的环境变量是不断累加的，参考 [Kubernetes中的服务发现与docker容器间的环境变量传递源码探究](https://jimmysong.io/posts/exploring-kubernetes-env-with-docker/)。为了更新容器中使用 ConfigMap 挂载的配置，需要通过滚动更新 pod 的方式来强制重新挂载 ConfigMap。

### 参考

- [Kubernetes 1.7 security in practice](https://acotten.com/post/kube17-security)
- [ConfigMap | kubernetes handbook - jimmysong.io](https://jimmysong.io/kubernetes-handbook/concepts/configmap.html)
- [创建高可用ectd集群 | Kubernetes handbook - jimmysong.io](https://jimmysong.io/kubernetes-handbook/practice/etcd-cluster-installation.html)
- [Kubernetes中的服务发现与docker容器间的环境变量传递源码探究](https://jimmysong.io/posts/exploring-kubernetes-env-with-docker/)