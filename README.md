
Configurando un cluster Kubernetes HA en Bare Metal:
=================================================
**Instalacion DocKer**

$ apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

$add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial edge"

$ apt-cache madison docker-ce
$ apt-get install -y docker-ce=17.03.3~ce-0~ubuntu-xenial
	NOTA: se puede instalar la version que se necesite
	
**Instalación de Kubernetes**
```
	$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
  sudo apt-get update -q

	$ apt-get install -y kubeadm=1.9.11-00 kubelet=1.9.11-00 kubectl=1.9.11-00 kubernetes-cni=0.6\*

```
**Requirements:**

 - Mínimo de tres nodos de Ubuntu (estos serían nodos maestros)
    - Esto manejará al menos una falla maestra. Para manejar dos fallas maestras, necesitamos cinco nodos maestros.
 - Nodos de trabajo (opcional puede ser 0 a n)
 - IP VIRTUAL (para  keepalived)

**Setup:**

1. Preparar los sistemas:
	- Desactivar intercambio en todos los nodos (sudo swapoff -a)
	- Apague el firewall si está habilitado.
	- Si estos nodos son nuevos, instale los paquetes Docker y Kubernetes.

2. Ejecuta etcd como un contenedor docker en cada nodo master.
	- Editar etcd/initetcd.sh si quiere agregar sus propias direcciones, con las direcciones IP (HOST_1, HOST_2, HOST_3)  y nombres (NAME_1, NAME_2, NAME_3) de los tres nodos.
		NOTA: por defecto tiene las IP´s (10.0.0.21,10.0.0.22,10.0.0.23) correspondientes a (kub01,kub02,kub03).
		      La carpeta "etcd" incluye tres archivos llamados (initk01.sh, initk02.sh, initk03.sh) para cada nodo master
	- Copia y ejecuta el script en cada uno de los tres maestros..
          	NOTA: En cada nodo, los valores (NODE_NAME & NODE_IP) en el script deben modificarse para incluir el nombre de host correcto (NAME_1, NAME_2, NAME_3) e IP (HOST_1, HOST_2, HOST_3).
			NOTA: por defecto tiene tres archivos que se ejecutan segun los nodos maestros, ya tiene configurado las ip y los nombres
	- Esto iniciará el contenedor docker etcd en los tres nodos..
	- Para probar la instalación. (para ver si los tres nodos son miembros del cluster etcd):
	  $ docker exec etcd /bin/sh -c "export ETCDCTL_API=3 && /usr/local/bin/etcdctl member list"

3. Ejecute la configuración de inicio de Kubeadm para configurar el primer maestro "solo ejecutar en el primer maestro".
	- Edite el archivo kubeClusterInitConfig/kubeadm-init.yaml con las direcciones IP y los nombres de los tres nodos. 
          NOTA: la IP virtual se debe agregar al campo apiServerCertSANs junto con los nodos maestros IP y nombres.
		  NOTA: por defecto ya tiene las IP´S y NOMBRES de los tres master (10.0.0.21,10.0.0.22,10.0.0.23).
	- Run:
	  ```
	  $ sudo kubeadm init --ignore-preflight-errors=all --config kubeadm-init.yaml
	  ```
	- Una vez completado, siga las instrucciones proporcionadas como parte de la salida para crear la carpeta .kube, copiando la configuración, etc.
	```	
		$ mkdir -p $HOME/.kube
		$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
		$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
	```
	- Apliar la configuración de red de flannel:
	```
		$ sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
	```
	- Run:
		```
		$ kubectl get nodes
		$ kubectl get pods --all-namespaces -o=wide
		```
		
	Asegúrese de que haya un maestro y que todos los pods estén funcionando bien. Si el nodo es NotReady (lo ideal es que esté listo), espere un tiempo para que esté listo. Lo mismo con los pods también, asegúrese de que todos los pods se estén ejecutando sin ningún problema antes de iniciar el otro maestro.
	
Si después de esperar unos minutos, el maestro aún está en el estado de No Preparado o los pods se están reiniciando o no todos están en el estado de Ejecución, algo está mal. Intente reiniciar el kubelet (sudo service kubelet restart). 

	- Copie los certificados kubernetes del primer maestro (aquel en el que el comando de inicio anterior se ejecutó con éxito) al segundo y tercer nodo maestro.
	```
	-----CREAR LA CARPETA /kpi
	$ mkdir /home/kub02/pki
	$ mkdir /home/kub03/pki

	-----EJECUTE ESTOS COMANDOS PARA COPIAR LOS pki
	
	$ sudo scp /etc/kubernetes/pki/* root@10.0.0.22:/home/kub02/pki/
	$ sudo scp /etc/kubernetes/pki/* root@10.0.0.23:/home/kub03/pki/
	```	
	---Si presenta problemas con la copia, una de las soluciones seria cambiar la clave de ssh de root con el siguiente comando

	$ passwd root
		--agregando la nueva clave de root para ssh, luego cambiando el acceso en el archivo
	$ nano /etc/ssh/sshd.config
		-- permitiendo el acceso a root
		-- descomentar PermitRootLogin Yes
		-- reiniciar sshd
	$ systemctl reload sshd

	- Copie la configuración (kubeClusterInitConfig/kubeadm-init.yaml) a los tres nodos y modifique el Nombre del nodo en el yaml al nombre del nodo donde se debe ejecutar la configuración.
		NOTA: por defecto aparece tres archivos llamados (kubeadm-init.yaml,kubeadm-init_2.yaml y kubeadm-init_3.yaml) con las direcciones ip configuradas y el nombre de los nodos.

		- Mueva los certificados pki /etc/kubernetes/pki y ejecute kubeadm init en los otros nodos:
		  ```
		  NOTA: copiarlo desde la carpeta pki del nodo maestro 1

		  $ sudo mv pki/* /etc/kubernetes/pki/
		  $ sudo kubeadm init --ignore-preflight-errors=all --config kubeadm-init.yaml
		  ```
	  	NOTA: Cuando se ejecuta kubeadm init, la salida generada difiere de la salida generada cuando init se ejecutó en el primer nodo. Esto es lo esperado.

		Una vez completado, siga las instrucciones proporcionadas como parte de la salida para crear la carpeta .kube, copiando la configuración, etc, en cada uno de los maestros 02 y 03
		```
		$ mkdir -p $HOME/.kube
		$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config	          
		$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
		```
		NOTA: la configuración de red (flannel) debe aplicarse solo en el primer nodo maestro. No necesitamos aplicarla de nuevo en los siguientes másters.
         	- Run:
	 	```
		$ kubectl get nodes	
		NAME             STATUS    ROLES     AGE       VERSION
		k8-node-a   Ready     master    1h        v1.11.1
		k8-node-b   Ready     master    1h        v1.11.1
		k8-node-c   Ready     master    1h        v1.11.1
		
		$ kubectl get pods --all-namespaces -o=wide
		NAMESPACE     NAME                                     READY     STATUS    RESTARTS   AGE       IP               NODE
		kube-system   coredns-78fcdf6894-w7wtf            1/1       Running   0          1h        10.244.0.7       k8-node-a
		kube-system   coredns-78fcdf6894-zztcm            1/1       Running   0          1h        10.244.0.5       k8-node-a
		kube-system   kube-apiserver-k8-node-a            1/1       Running   0          1h        10.244.120.123   k8-node-a
		kube-system   kube-apiserver-k8-node-b            1/1       Running   0          1h        10.244.120.124   k8-node-b
		kube-system   kube-apiserver-k8-node-c            1/1       Running   0          1h        10.244.120.164   k8-node-c
		kube-system   kube-controller-manager-k8-node-a   1/1       Running   0          1h        10.244.120.123   k8-node-a
		kube-system   kube-controller-manager-k8-node-b   1/1       Running   0          1h        10.244.120.124   k8-node-b
		kube-system   kube-controller-manager-k8-node-c   1/1       Running   0          1h        10.244.120.164   k8-node-c
		kube-system   kube-flannel-ds-76q6s               1/1       Running   0          1h        10.244.120.124   k8-node-b
		kube-system   kube-flannel-ds-8nsjn               1/1       Running   0          1h        10.244.120.164   k8-node-c
		kube-system   kube-flannel-ds-vpxkf               1/1       Running   0          1h        10.244.120.123   k8-node-a
		kube-system   kube-proxy-cqmm6                    1/1       Running   0          1h        10.244.120.124   k8-node-b
		kube-system   kube-proxy-lffj5                    1/1       Running   0          1h        10.244.120.123   k8-node-a
		kube-system   kube-proxy-w2sn2                    1/1       Running   0          1h        10.244.120.164   k8-node-c
		kube-system   kube-scheduler-k8-node-a            1/1       Running   0          1h        10.244.120.123   k8-node-a
		kube-system   kube-scheduler-k8-node-b            1/1       Running   0          1h        10.244.120.124   k8-node-b
		kube-system   kube-scheduler-k8-node-c            1/1       Running   0          1h        10.244.120.164   k8-node-c
		
		```
		Esto mostrará el número de maestros y el estado de los pods. Es posible que pasen unos minutos para que todos los pods estén en funcionamiento y los nodos se vuelvan "Preparados".
		Algunos de los pods se reinician una o dos veces, lo que debería estar bien. Pero, no deben ser de bucle de loop.

4. Instalar Keepalived.
	- Seguir keepalived/README para configurarlo.

5. Instalar nginx equilibrador de carga para acceder al servidor api.
	- Seguir nginx-lb/README para configurarlo.

6. Para agregar trabajadores al cluster:
	
	- En uno de los maestros, puede ser cualquier maestro:
		```
		$ sudo kubeadm token create
		3k8a5g.aue65i8z54z4xd7x
		
		$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
		4908ea01832102792d641f3359efbfc8dfc64ad924128be0484f88b3933a6d27
		```
		- En el nodo trabajador:
	
		```
		$ sudo kubeadm join <master-ip>:6443 --token 3k8a5g.aue65i8z54z4xd7x --discovery-token-ca-cert-hash sha256:4908ea01832102792d641f3359efbfc8dfc64ad924128be0484f88b3933a6d27
		```
		
		Después de que la unión se complete con éxito:
		```
		$ sed -i "s/10.0.0.21:6443/10.0.0.28:16443/g" /etc/kubernetes/bootstrap-kubelet.conf
		$ sed -i "s/10.0.0.22:6443/10.0.0.28:16443/g" /etc/kubernetes/bootstrap-kubelet.conf
		$ sed -i "s/10.0.0.23:6443/10.0.0.28:16443/g" /etc/kubernetes/bootstrap-kubelet.conf
		
		$ sed -i "s/10.0.0.21:6443/10.0.0.28:16443/g" /etc/kubernetes/kubelet.conf
		$ sed -i "s/10.0.0.22:6443/10.0.0.28:16443/g" /etc/kubernetes/kubelet.conf
		$ sed -i "s/10.0.0.23:6443/10.0.0.28:16443/g" /etc/kubernetes/kubelet.conf

		$ grep 10.0.0.28 /etc/kubernetes/*.conf 
		/etc/kubernetes/bootstrap-kubelet.conf:    server: https://10.0.0.28:16443
		/etc/kubernetes/kubelet.conf:    server: https://10.0.0.28:16443

		$ sudo systemctl restart docker kubelet
		```
	- En el maestro, puede er cualquiera:
		- Run:
		```
		  $ kubectl get nodes
		```
		Para ver si el nodo de trabajo se ha añadido correctamente. Es posible que no tenga una etiqueta ('worker') que se pueda configurar manualmente más adelante.

7. Para hacer programable el nodo maestro.
	- Run:
	```
		$ kubectl taint nodes --all node-role.kubernetes.io/master-
	```
	- Para obtener la lista de nodos que son programables:
	```
		$ kubectl get nodes -o go-template='{{range .items}}{{with $x := index .metadata.annotations "scheduler.alpha.kubernetes.io/taints"}}{{.}}{{end}}{{end}}'
	```
8. Para probar el cluster:

	- En cualquier maestro:
   	
	```
	$ kubectl run nginx --image=nginx --replicas=3 --port=80
	deployment.apps/nginx created

	$ kubectl get pods -l=run=nginx -o wide
	NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
	nginx-6f858d4d45-mdtnw   1/1       Running   0          11s       10.244.2.2   k8-node-c
	nginx-6f858d4d45-nzsdm   1/1       Running   0          11s       10.244.1.2   k8-node-b
	nginx-6f858d4d45-x77w4   1/1       Running   0          11s       10.244.0.8   k8-node-a

	$ kubectl expose deployment nginx --type=NodePort --port=80
	service/nginx exposed

	$ kubectl get svc -l=run=nginx -o wide
	NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE       SELECTOR
	nginx     NodePort   10.100.190.219   <none>        80:30908/TCP   9s        run=nginx
		
	NOTA: 
		1. https no funciona ya que nginx ejecuta solo http
		2. El número de puerto utilizado a continuación en el comando curl es del campo PORT (S) en la salida del comando anterior..

	$ curl http://10.0.0.28:30908
	<!DOCTYPE html>
	<html>
	<head>
	<title>Welcome to nginx!</title>
	<style>
	    body {
	        width: 35em;
	        margin: 0 auto;
	        font-family: Tahoma, Verdana, Arial, sans-serif;
	    }
	</style>
	</head>
	<body>
	<h1>Welcome to nginx!</h1>
	<p>If you see this page, the nginx web server is successfully installed and
	working. Further configuration is required.</p>
	
	<p>For online documentation and support please refer to
	<a href="http://nginx.org/">nginx.org</a>.<br/>
	Commercial support is available at
	<a href="http://nginx.com/">nginx.com</a>.</p>
	
	<p><em>Thank you for using nginx.</em></p>
	</body>
	</html>
	$
	```
	
	- Prueba de conectividad POD:
	
	```
		$ kubectl run nginx-server --image=nginx --port=80
		deployment.apps/nginx-server created

		$ kubectl expose deployment nginx-server --port=80
		service/nginx-server exposed

		$ kubectl get pods -o wide -l=run=nginx-server
		NAME                            READY     STATUS    RESTARTS   AGE       IP           NODE
		nginx-server-6bdb9998f7-fk54d   1/1       Running   0          16s       10.244.2.3   k8-node-c

		$ kubectl run nginx-client -ti --rm --image=alpine -- ash
		Si no ve un indicador de comando, intente presionar enter.

		/ # wget nginx-server
		Connecting to nginx-server (xxx.xxx.xxx.xxx:80)
		index.html           100% |*******************************************|   612   0:00:00 ETA

		/ # cat index.html
		<!DOCTYPE html>
		<html>
		<head>
		<title>Welcome to nginx!</title>
		<style>
		    body {
		        width: 35em;
		        margin: 0 auto;
		        font-family: Tahoma, Verdana, Arial, sans-serif;
		    }
		</style>
		</head>
		<body>
		<h1>Welcome to nginx!</h1>
		<p>If you see this page, the nginx web server is successfully installed and
		working. Further configuration is required.</p>
		
		<p>For online documentation and support please refer to
		<a href="http://nginx.org/">nginx.org</a>.<br/>
		Commercial support is available at
		<a href="http://nginx.com/">nginx.com</a>.</p>
		
		<p><em>Thank you for using nginx.</em></p>
		</body>
		</html>
		/ #
		/ # la sesión finalizó, continúe con el comando 'kubectl attach nginx-client-65975cddb-fcnf2 -c nginx-client -i -t' cuando el pod se esté ejecutando
		deploy.apps "nginx-client" eliminado

		$ kubectl delete deploy,svc nginx-server
		deployment.extensions "nginx-server" deleted
		service "nginx-server" deleted
		```

### Solución de problemas: ###


1. Cómo reiniciar el sistema si algo sale mal:
	- Retire el equilibrador de carga nginx si está instalado. Siga la sección de Solución de problemas en nginx-lb/README.
	- Ejecute kubeClusterInitConfig/cleanup.sh en todos los nodos.
	```
		$ sudo sh cleanup.sh
	```

2. Al reiniciar los nodos maestros:
	- Asegúrese de que se estén ejecutando el etcd, keepalived y el nginx-lb. Si no se está ejecutando, inicie el contenedor antiguo manualmente.
	- Compruebe si el intercambio todavía está desactivado. De lo contrario apágalo usando 'sudo swapoff -a'.
	- Reinicie el kubelet si se realiza una o las dos operaciones anteriores.
	```
		$ sudo service kubelet restart
	```

NOTA:
	Este manual fue tomado desde https://github.com/kmyn/k8hacluster, dueño de estas instrucciones, se acoplo a la ejecucion del proyecto de Sistemas Operativos 2 de UMG San Jose Pinula