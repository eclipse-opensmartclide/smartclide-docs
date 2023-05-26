# For the sysadmin
<div style="text-align: justify;">
	<p>
		This page contains some very-easy-to-follow guides for sysadmins that want to set up and deploy an on-premises instance of the SmartCLIDE IDE.
	</p>
</div>

## SmartCLIDE Helm Chart

### Requirements

- A publicly resolvable domain, like `example.org` or a subdomain like `foo.example.org`.

  You **must** be able to edit DNS records for that domain.

- A wildcard TLS certificate for your domain, i.e. for `*.example.org` or `*.foo.example.org`.

  It **must** be a wildcard certificate! Letsencrypt is **not** supported at the moment!
  Save the certificate files in `smartclide.full-chain.crt` and `smartclide.key`.

  `smartclide.full-chain.crt` **must** contain the full certificate chain, including intermediate certificates.

- An SMTP server
- A GitLab instance or an account on gitlab.com
- A GitHub account
- A [kubernetes](https://kubernetes.io/) cluster (version 1.21+) with
  - Ingress Controller
  - Dynamic Volume Provisioning
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) installed and configured for your cluster
- [helm](https://helm.sh/) installed and configured for your cluster

**Disclaimer:** the above tools can be installed on both Linux and Windows.
However, the following instructions have only been tested on Linux.

### Prepare

#### DNS

Create a `CNAME` record for `*.<YOUR_DOMAIN>` - e.g. `*.example.org` - and set the target to the output of:

```shell
kubectl get services --namespace ingress-nginx -o jsonpath="{.items[].status.loadBalancer.ingress[0].hostname}"
```

Replace `ingress-nginx` with the namespace where your ingress controller is installed.

#### Eclipse Che

##### Install chectl

- Download `chectl` version **`7.38.0`** for your operating system
  from the [chectl release page](https://github.com/che-incubator/chectl/releases/tag/7.38.0), e.g. `chectl-linux-x64.tar.gz`.
- Unpack it to your `HOME` folder:

  ```shell
  tar -xvz -C ${HOME} -f chectl-linux-x64.tar.gz
  ```

- Create an alias for the chectl command:

  ```shell
  chectl=${HOME}/chectl/bin/run
  ```

##### Namespace

Create the namespace for Eclipse Che:

```shell
kubectl create namespace eclipse-che
```

##### Certificate

Create the TLS certificate secret for Eclipse Che:

```shell
kubectl create secret tls che-tls \
  --namespace eclipse-che \
  --cert=smartclide.full-chain.crt \
  --key=smartclide.key
```

##### Deploy Eclipse Che

1. Configure deployment scripts. Run:

   ```shell
   sed -i s/"{THE_DOMAIN}"/"<YOUR_DOMAIN>"/g che/*.{yaml,json}
   ```

   Replace `<YOUR_DOMAIN>` with your domain, e.g. `example.org`.

2. Create initial deployment. Run:

   ```shell
   chectl server:deploy \
      --installer=helm \
      --platform=k8s \
      --chenamespace=eclipse-che \
      --domain=<YOUR_DOMAIN> \
      --telemetry=off \
      --no-auto-update \
      --multiuser \
      --helm-patch-yaml=che/my-values.yaml
   ```

   Replace `<YOUR_DOMAIN>` with your domain, e.g. `example.org`.

   Answer with `n` ("no") when asked `Do you want to update chectl now? [y/n]:`.

   Answer with `y` ("yes") when asked `'helm' installer is deprecated. Do you want to proceed? [y/n]:`.

   The command will take a while. At the end you should see a message like:

   ```shell
       ✔ Eclipse Che 7.38.0 has been successfully deployed.
       ✔ Documentation             : https://www.eclipse.org/che/docs/
       ✔
   -------------------------------------------------------------------------------
       ✔ Users Dashboard           : https://che.example.org
       ✔ Admin user login          : "admin:admin". NOTE: must change after first login.
       ✔
   -------------------------------------------------------------------------------
       ✔ Plug-in Registry          : https://plugin-registry-eclipse-che.example.org/v3/
       ✔ Devfile Registry          : https://devfile-registry-eclipse-che.example.org/
       ✔
   -------------------------------------------------------------------------------
       ✔ Identity Provider URL     : https://keycloak-eclipse-che.example.org/auth/
       ✔ Identity Provider login   : "admin:s3cr3t".
       ✔
   -------------------------------------------------------------------------------
   Command server:deploy has completed successfully in 02:34.
   ```

3. Update Eclipse Che configuration. Run:

   ```shell
   kubectl patch configmaps --namespace eclipse-che che --patch "$(cat che/config-patch.yaml)" -o yaml

   for theIngress in che-ingress che-dashboard-ingress devfile-registry plugin-registry keycloak-ingress
   do
     kubectl patch ingress --namespace eclipse-che ${theIngress} --patch "$(cat che/ingress-patch.yaml)" -o yaml
   done

   kubectl --namespace eclipse-che rollout restart deployment/che
   ```

4. Open Che Users Dashboard `https://che.<YOUR_DOMAIN>`, e.g. `https://che.example.org`, in your browser, and login with the
   admin username and password provided in the output of `chectl server:deploy` command above.
   Change the default admin password to a secure password!

5. Open keycloak admin interface `https://keycloak-eclipse-che.<YOUR_DOMAIN>/auth/admin/`,
   e.g. `https://keycloak-eclipse-che.example.org/auth/admin/`, in your browser, and login with the keycloak admin username and
   password provided in the output of `chectl server:deploy` command above.

   - Make sure that the realm `Che` is selected at the top of the left-hand menu.
   - Under the menu section `Manage`, click on `Import` and select [che/realm-patch.json](che/realm-patch.json).
     Make sure that `Import clients` and `Import realm roles` is `ON`. Click `Import`.
     You should see a notification that 12 records have been added.
   - Under the menu section `Configure`, click on `Realm Settings`, then click on the tab `Login` and change `Require SSL`
     to `external requests`. Click `Save`.
   - Click on the tab `Email` and fill in the values for your SMTP server, so that keycloak can send passwort reset emails.
     Click `Save`.
   - Under the menu section `Configure`, click on `Roles` and the tab `Default Roles`. In the `Available Roles` box, select

     - developer
     - kie-sever
     - rest-all
     - user

     Click `Add selected`.

   - Under the menu section `Configure`, click on `Clients`, and select `business-central`. Click on the tab `Credentials` and
     then on `Regenerate Secret`. Note down the new secret.

     Repeat this for the client `kie-server`.

#### Dedicated Node Group for DLE

The DLE component requires more resources than other SmartCLIDE components.
The minimum requirements are _4 CPUs_ and _16 GiB RAM_.
It is therefore recommended to create a dedicated node group for the DLE with at least 1 node of the required minimum size.

Add the following `label` and `taint` to the node group / **all** nodes in the node group:

```yaml
labels:
  smartclide-nodegroup-type: "dle"
```

```yaml
taints:
  - key: "dle"
    value: "true"
    effect: "NoSchedule"
```

#### Settings

Open [my-values.yml](my-values.yaml) in a text editor and change the values according to your needs.
See the comments in the file for more information.

#### Namespace

Create the namespace:

```shell
kubectl create namespace smartclide
```

#### Certificate

Create the TLS certificate secret:

```shell
kubectl create secret tls smartclide-tls \
  --namespace smartclide \
  --cert=smartclide.full-chain.crt \
  --key=smartclide.key
```

### Install / Upgrade

```shell
helm upgrade --install --namespace smartclide --values my-values.yaml smartclide helm-chart
```

### Uninstall

```shell
helm uninstall --namespace smartclide smartclide
```
