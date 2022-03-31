## Interact with Kubernetes resources in PHP 8

I’ve been using Kubernetes for about 4 years now, and I was amazed so much by the fact that there is so much versatility from any programming language when using Kubernetes. Any single interface that can make HTTP requests will talk smoothly with any Kubernetes cluster.

One of the problems I faced was to be able to communicate with my Kubernetes clusters in PHP. GO is a still known programming language in which you may find tons of controllers written for Kubernetes, from ingress controllers to new CRDs that enable special features, like Agones or Traefik. However, this wasn’t viable for me at that time, meaning that I had to learn GO and master it, so then I could build controllers for my clusters.

The other alternative would have been to choose any already-built PHP libraries, but many of them lacked proper functional tests, consistency, and ease of use for custom resources, like adding your own methods for each class or even building your own CRD classes without too much hassling.

This is how PHP K8S was born: a PHP-based client that eases the use of writing controllers or managing a Kubernetes cluster from the outside in PHP, with minimal configuration and with an extensive API that makes customization a no-hassle job.

- [Github](https://github.com/renoki-co/php-k8s)

<hr>

For a brief example, we will turn this Service configuration into PHP OOP code:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

The YAML can be transposed to this:

```php
use RenokiCo\PhpK8s\KubernetesCluster;

// Create a new instance of KubernetesCluster.
$cluster = new KubernetesCluster('http://127.0.0.1:8080');

// Create a new NGINX service.
$svc = $cluster->service()
    ->setName('nginx')
    ->setNamespace('frontend')
    ->setSelectors(['app' => 'frontend'])
    ->setPorts([
        ['protocol' => 'TCP', 'port' => 80, 'targetPort' => 80],
    ])
    ->create();
```

### Extending into CRDs

Besides the basic functionalities, Kubernetes gives you the  [ability to create custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)  via the so-called Custom Resource Definitions (or CRDs). Custom Resources can also be easily created for PHP K8s. However, the CRDs should already be defined in your cluster before being able to create a new PHP K8s class, so that you will have a name and a group version for it. Soon, you will be able to create Custom Resource Definitions from PHP K8s directly.

Take the following Agones Game Server CRD as an example:  [apiextensions.k8s.io/v1](https://github.com/googleforgames/agones/blob/v1.13.0/install/helm/agones/templates/crds/gameserver.yaml) . Once this CRD exists within your cluster, PHP K8s is now your playground:

```php
use RenokiCo\PhpK8s\Contracts\InteractsWithK8sCluster;
use RenokiCo\PhpK8s\Kinds\K8sResource;
use RenokiCo\PhpK8s\Traits\HasSpec;

class GameServer extends K8sResource implements InteractsWithK8sCluster
{
    use HasSpec;
    
    /**
     * The resource Kind parameter.
     *
     * @var null|string
     */
    protected static $kind = 'GameServer';

    /**
     * The default version for the resource.
     *
     * @var string
     */
    protected static $defaultVersion = 'agones.dev/v1';

    /**
     * Wether the resource has a namespace.
     *
     * @var bool
     */
    protected static $namespaceable = true;
}
$gs = new GameServer($cluster);
$gs->setName('minecraft')
   ->setSpec(['template' => ...])
   ->create();
```

For further reference, read more about  [how to extend your own CRDs](https://github.com/renoki-co/php-k8s/blob/master/docs/CUSTOM-CRDS.md) , including how to use custom traits that add shared functionalities between other resources or make your resource scalable, loggable, podable (manages pods; i.e. Jobs or Deployments), or watchable.


### Laravel Macros

We tried to make PHP K8s as customizable as possible. Besides having custom callers to be able to add or remove any field without submitting a PR for your specific case, you can also make use of Laravel Macros: a powerful feature to create custom functions that you can call in your code.

The way Macros work is within the call magic methods. When defining a macro, the call is stored, and whenever the function name gets encountered and it doesn’t exist within the code, it also searches in the stored methods before throwing an error.

Macros are really useful when interacting with the cluster. In the previous example, you have seen how you can create a custom class and how to initialize it, but the recommended way is to create a KubernetesCluster method that can be called directly from the object, rather than passing the cluster as a parameter for initializing the class. We can do these using Macros:

```php
use RenokiCo\PhpK8s\K8s;

K8s::macro('gameServer', function ($cluster = null, array $attributes = []) {
    return new Path\To\Class\GameServer($cluster, $attributes);
});

$gs = $cluster->gameServer() // call directly from $cluster
   ->setName('minecraft')
   ->setSpec(['template' => ...])
   ->create();
```

### Pod Exec API

One of the most recent features is Pod Exec. Pod Exec allows you to execute commands in pod containers directly from the kubectl interface. Usually, you would do the following to connect to a busybox-pod pod, in the busybox container:

```bash
kubecl exec -it busybox-pod -c busybox -- /bin/sh
```

One of the most challenging features was to be able to communicate with WebSockets to send commands to the Kubernetes API as kubectl exec does.

Since PHP is a block-IO programming language, this is hard to achieve WebSocket connections. Underneath the feature, ratchet/pawl is used to open a sync loop using ReactPHP and wait for the command to finish:

```php
$messages = $pod->exec(['/bin/sh', '-c', 'ls -al']);

foreach ($messages as $message) {
    echo "[{$message['channel']}] {$message['output']}".PHP_EOL;
}
```

<hr>

I Hope PHP K8s unlocked your potential ideas of interacting with any Kubernetes cluster and will seem to be a good competitor for being adopted as a major player in the game of Kubernetes clients.
Please open issues or feature requests within the Issues board on Github or if you think there are some improvements needed, submit a Pull Request and we will discuss it there.

## 💸 Sponsorship

Hi, I'm [Alex](https://github.com/rennokki), the founder of [Renoki Co.](https://github.com/renoki-co). I'm thankful for taking your time to read this article, and I hope that it helped you. Developing and maintaining packages and delivering good articles about Laravel, Kubernetes and AWS take a lot of time, but I believe it's a time well spent.

If you support more helpful articles, or you are using one or more Renoki Co. open-source packages in your production apps, in presentation demos, hobby projects, school projects or so, sponsor our work with [Github Sponsors](https://github.com/sponsors/rennokki). 📦
