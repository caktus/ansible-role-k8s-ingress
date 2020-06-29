# caktus.k8s-web-cluster

An Ansible role to help configure Kubernetes clusters for web apps.

Supported cloud providers include GCP (GKE), AWS (EKS), Azure (AKS), and Digital
Ocean. The configuration includes installing:

* Nginx Ingress Controller
* Certificate manager (https://cert-manager.io/docs/) (https://github.com/jetstack/cert-manager)
* Let's Encrypt certificate issuer
* Logspout for Papertrail
* For AWS, granting cluster access to IAM users


# License

This Ansible role is released under the BSD License. See the
[LICENSE](https://github.com/caktus/ansible-role-k8s-web-cluster/blob/master/LICENSE)
file for more details.

Development sponsored by [Caktus Consulting Group, LLC](http://www.caktusgroup.com/services>).


# Requirements

* ``pip install openshift kubernetes-validate``


# Installation

1. Add to your ``requirements.yaml``:


```yaml
---
# file: deploy/requirements.yaml

- src: https://github.com/caktus/ansible-role-k8s-web-cluster
  version: init-role  # TODO: remove
  name: caktus.k8s-web-cluster
```

2. Add the role to your playbook:

```yaml
---
# file: deploy/deploy.yaml

- hosts: k8s
  roles:
    - role: caktus.k8s-web-cluster
```

3. Add role vars configuration:

```yaml

---
# ----------------------------------------------------------------------------
# caktus.k8s-web-cluster: Configure kubernetes cluster for web apps
# ----------------------------------------------------------------------------

k8s_cluster_type: <aws|gcp|azure|digitalocean>
k8s_context: <name of context from ~/.kube/config>
k8s_cluster_name: <display name for your cluster>
k8s_letsencrypt_email: <email to contact about expiring certs>
k8s_echotest_hostname: <test hostname assigned to your cluster ip, e.g. echotest.caktus-built.com>
# aws only:
k8s_iam_users: [list of IAM usernames who should be allowed to manage the cluster]
```

4. Run ``deploy.yaml`` playbook:

```sh
ansible-playbook -l <host/group> deploy.yaml -vv
```


### Testing that Let's Encrypt is working

1. Find the hostname or IP of your load balancer.
2. Add a DNS record for ``k8s_echotest_hostname`` to
   point to this hostname or IP address (switching the record type if needed).
3. Give the record a minute or two to propagate.
4. Add an echotest playbook:

```yaml
---
# file: echotest.yaml

- hosts: k8s
  tasks:
    - name: Install echo test server
      import_role:
        name: caktus.k8s-web-cluster
        tasks_from: echotest
```

5. Run ``echotest.yaml`` playbook:

```sh
ansible-playbook -l <host/group> echotest.yaml -vv
```

6. Give the certificate a couple minutes to be generated and validated. While waiting,
   you can watch the output of:

       kubectl -n echoserver get pod

   When the ``cm-acme-http-solver`` pod goes away, the certificate should be
   validated. Now, navigate to ``k8s_echotest_hostname`` and ensure that you
   have a valid certificate. If you don't, you want to follow the
   [cert-manager troubleshooting](https://docs.cert-manager.io/en/latest/getting-started/troubleshooting.html)
   steps in the documentation. But, be sure to reload a few times, and close the
   browser tab and open a new one to make sure it's really broken, because
   sometimes it takes a few minutes to go through and the browser gets stuck
   with the temporary certificate.

7. You should see the ``*-tls`` secret in the **echoserver** namespace:

       kubectl -n echoserver get secret
       NAME                  TYPE                                  DATA   AGE
       default-token-62pdt   kubernetes.io/service-account-token   3      5m
       echoserver-tls        kubernetes.io/tls                     3      5m

   If not, you may need to re-create the ingress by deleteing and re-applying
   it.

8. When you're done, delete the echotest resources from the cluster. Run:

```sh
ansible-playbook -l <host/group> echotest.yaml --extra-vars "k8s_echotest_state=absent" -vv
```
