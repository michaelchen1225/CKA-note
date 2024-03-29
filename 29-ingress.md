# ingress

https://kubernetes.io/docs/concepts/services-networking/ingress/

Now, in k8s version 1.20+ we can create an Ingress resource from the imperative way like this:-

Format - kubectl create ingress <ingress-name> --rule="host/path=service:port"

Example - kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"

Find more information and examples in the below reference link:-

https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-ingress-em-

References:-

https://kubernetes.io/docs/concepts/services-networking/ingress

https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types

https://tsunghsien.gitbooks.io/kubenetes/content/ingresspei-zhi.html


## Ingress controller :
https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/learn/lecture/21738990#overview

## Ingress - Annotations and rewrite-target

解釋:
https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/learn/lecture/16827080#overview


controlplane ~ ➜  k logs -n critical-space services/pay-service 
 * Serving Flask app 'app' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://10.244.0.11:8080/ (Press CTRL+C to quit)
10.244.0.9 - - [27/Mar/2024 06:00:55] "GET /pay HTTP/1.1" 404 - # 沒加annotations
10.244.0.9 - - [27/Mar/2024 06:03:45] "GET / HTTP/1.1" 200 - # 加了annotations