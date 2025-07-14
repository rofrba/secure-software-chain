## 🚀 ¡Protege tus Pipelines CI/CD con Sigstore y Kyverno! 🔒
### Firmando y Verificando tus Imágenes de Contenedor
¡Hola a todos los entusiastas de la seguridad y el desarrollo! 👋 
Hoy vamos a meternos de lleno en un tema súper importante para cualquiera que trabaj econ contenedores y Kubernetes. ¿Cómo nos aseguramos de que las imágenes que desplegamos en nuestros clustes son de confianza? Algo que nos puede ayudar en esta pregunta es __firmar nuestras imágenes de contedores__


### 💡 ¿Por Qué Firmar tus Imágenes Es Fundamental?
Imagina tu cadena de suministro de software como una receta secreta. Si no verificas la procedencia de cada ingrediente (tus imágenes de contenedor), ¡podrías terminar con un plato desastroso o incluso peligroso! 😱

Con el auge de DevOps y la velocidad de lanzamiento, la seguridad se ha vuelto una prioridad. ¿Cómo evitamos que alguien introduzca código malicioso en nuestra cadena de suministro de software? Firmar tus imágenes es la respuesta. Así, ponemos un __sello de aprobado__ digital a todo lo que construimos, asegurando que solo el software verificado llegue a nuestros entornos. ¡Es la esencia de DevSecOps: integrar la seguridad desde el principio!

### 🦸‍♂️ Los Héroes de la Seguridad: Sigstore y Kyverno
#### 🔐 Sigstore: Tu Notario Digital para el Software
Sigstore es una iniciativa asombrosa que nos brinda una forma abierta, transparente y accesible de asegurar la cadena de suministro de software. Piensa en ella como tu certificado de autenticidad para el código. Con Sigstore puedes:
- Firmar, verificar y proteger tu software con una firma digital.

- Rastrear el software hasta su origen, dándote total confianza de que no ha sido alterado.

- Generar los pares de claves (pública y privada) necesarios para firmar y verificar tus artefactos.

Una de las características más geniales de Sigstore es su tecnología de registro transparente. Esto significa que cualquiera puede encontrar y verificar las firmas. Si alguien intenta modificar tu código fuente, artefacto o plataforma de compilación, ¡te enterarás al instante! 🕵️‍♀️


#### 🛡️ Kyverno: El Guardián de tus Clústeres de Kubernetes
Una vez que hemos firmado nuestras imágenes, el siguiente paso es garantizar que solo las imágenes con firmas válidas y verificadas puedan desplegarse en nuestros clústeres. ¡Aquí es donde Kyverno brilla!

Kyverno es un motor de políticas diseñado específicamente para Kubernetes. ¿Lo mejor? Sus políticas se gestionan como recursos de Kubernetes, ¡así que no necesitas aprender un nuevo lenguaje! Kyverno actúa como un controlador de admisión dinámico: intercepta las solicitudes de despliegue y aplica sus políticas para permitir o rechazar el despliegue.

Con Kyverno, puedes:

- Validar, mutar y generar recursos de Kubernetes.

- Gestionar políticas usando tus herramientas favoritas como kubectl, kustomize y git.

- Probar tus políticas como parte de tu pipeline CI/CD con su CLI.

Kyverno se apoya en Cosign (parte del proyecto Sigstore) para verificar las imágenes. En esta guía, nos centraremos en cómo firmar y verificar imágenes de contenedores usando Cosign y Kyverno. Para la demo, usaré Minikube, pero puedes usar cualquier clúster de Kubernetes.

### 🛠️ Demo time 
#### Requisitos Previos 
Antes de comenzar, asegúrate de tener lo siguiente:

- Conocimientos básicos de Contenedores y Kubernetes.

- Minikube y kubectl instalados.

- Cosign instalado.

- Kyverno instalado.

- Docker/Podman instalado.



#### 🔑 Generando tus Claves con Cosign
Lo primero es crear tu par de llaves: una privada (tu secreto) y una pública (para que el mundo verifique).

~~~ Bash

cosign generate-key-pair 
~~~

Este comando genera cosign.key (tu llave privada, ¡guárdala muy bien!) y cosign.pub (tu llave pública). Las claves privadas deben almacenarse de forma segura, usa herramientas como HashiCorp Vault o Amazon KMS para gestionarlas en tu CI/CD

#### 🏗️ Construyendo y Pusheando tu Imagen de Contenedor
Para esta demo, usaremos una imagen simple de Nginx.

~~~ Bash

podman build -t nginx-signed:v1 files
podman push [YOUR_REGISTRY]/nginx-signed:v1

~~~

#### ✍️ Firmando tu Imagen con Sigstore
🚨 Importante: Antes de firmar, la imagen debe estar publicada en un registro (como Docker Hub) y debes tener permisos de escritura. Cosign no firma imágenes que no estén en un registro.

Si intentas firmar sin permisos o una imagen local no publicada, Cosign te alertará:
~~~
UNAUTHORIZED: authentication required
accessing entity: entity not found in registry
~~~


Firmamos la imagen, utilizando el comando cosign sign, donde indicamos que utilice nuestra llave privada

~~~ Bash

cosign sign --key cosign.key quay.io/ralvarez1/nginx-signed:v1
~~~

Este comando creará una nueva etiqueta SHAxxxxx.sig en tu registro OCI, confirmando la firma.

#### ✅ Verificando tus Imágenes Firmadas
Para verificar la firma de una imagen, usaremos nuestra llave pública (cosign.pub):

~~~ Bash

cosign verify --key cosign.pub quay.io/ralvarez1/nginx-signed:v1 | jq -r .
~~~



💡 Recomendación clave: ¡Usa etiquetas inmutables en tus imágenes! Esto asegura que siempre obtengas la misma imagen. Mutar etiquetas puede ser útil en desarrollo, pero en producción, ¡es una fuente de riesgos!

#### ⚙️ Instalando Kyverno en tu Clúster de Kubernetes
Ahora, instalemos Kyverno en nuestro clúster (Minikube en este caso, pero es compatible con cualquier clúster Kubernetes):

Añade el repositorio Helm de Kyverno:

~~~ Bash

helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
~~~
Instala Kyverno con Helm:

~~~Bash

helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace
~~~
🚨 Importante: En ciertas versiones de k8s, particularmente en entornos locales, es necesario modificar una variable de Kyverno, para que pueda compartir red y registros DNS con el host. Por lo que es necesario utilizar un archivo values.yaml y modificar la entrada

~~~yaml
hostNetwork: true
~~~
Para crear la instancia de kyverno utilizando un values custom, ejecutamos el siguiente comando:
~~~bash
helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace -f Kyverno/values.yaml
~~~


Verifica que los pods de Kyverno estén en estado Running:

~~~Bash

kubectl get pods -n kyverno
~~~

#### 🛑 Verificando Imágenes en tu Clúster con Kyverno
Queremos asegurarnos de que solo las imágenes firmadas con nuestras claves puedan desplegarse en nuestro entorno, bloqueando cualquier imagen maliciosa o no confiable. Kyverno es nuestro guardián aquí.

Crearemos un recurso ClusterPolicy de Kyverno que se encargará de verificar las firmas de las imágenes de contenedor:

~~~YAML

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image
spec:
  validationFailureAction: Enforce 
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail
  rules:
    - name: check-image
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - image: "quay.io/ralvarez1/*" 
          key: |-
            -----BEGIN PUBLIC KEY-----
            YOUR-PUBLIC-KEY
            -----END PUBLIC KEY-----
~~~

Esta política exige que la validación se cumpla (Enforce) y que, en caso de fallo, el despliegue sea rechazado (Fail). La regla verifyImages verificará las firmas de las imágenes que coincidan (por eso el comodín *), usando la clave pública que generamos.
~~~ Bash
kubectl apply -f Kyverno/check-signature-policy.yaml
~~~

Ahora, probemos desplegando un pod con nuestra imagen firmada desde Docker Hub:

~~~ Bash

kubectl create namespace test
kubectl run nginx-signed --image quay.io/ralvarez1/nginx-signed:v1
~~~

Verás cómo la regla verifyImages de Kyverno verifica la imagen con la clave pública antes de que el contenedor se ejecute. 

#### 🚫 Probando la Política de Kyverno
¿Qué pasa si intentamos desplegar un pod con una imagen que no está firmada por nuestro par de claves de Cosign?

~~~ Bash

podman build -t quay.io/ralvarez1/nginx-unsigned:v1 .
podman push quay.io/ralvarez1/nginx-unsigned:v1
kubectl run nginx-unsigned --image quay.io/ralvarez1/nginx-unsigned:v1 -n test
~~~

¡Acceso Denegado! Recibirás un error similar a este:
~~~
Error from server (Admission webhook "validate.kyverno.svc-fail" denied the request):
policy check "check-image" (rule "check-image"):
  verifyImages: quay.io/ralvarez1/nginx-unsigned:v1
    failed: no matching signatures
~~~

¡Kyverno ha bloqueado el despliegue de una imagen no verificada! 🎉 


### Próximos pasos