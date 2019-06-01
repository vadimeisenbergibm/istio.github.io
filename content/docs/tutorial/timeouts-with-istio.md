---
title: Timeouts with Istio
overview: Mitigate faults related to delays by Istio.

weight: 120

---

In this module you deploy a faulty `ratings` microservice, which misbehaves half the time. With the probability of one
half it performs a delay of 7 seconds, all other times it succeeds.

To prevent user from waiting for 7 seconds, you use timeouts. It is often better to present partial information to the
user instead of letting the user wait for a long period of time.

1.  Deploy the faulty version of `ratings`:

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    # Copyright 2017 Istio Authors
    #
    #   Licensed under the Apache License, Version 2.0 (the "License");
    #   you may not use this file except in compliance with the License.
    #   You may obtain a copy of the License at
    #
    #       http://www.apache.org/licenses/LICENSE-2.0
    #
    #   Unless required by applicable law or agreed to in writing, software
    #   distributed under the License is distributed on an "AS IS" BASIS,
    #   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    #   See the License for the specific language governing permissions and
    #   limitations under the License.

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: ratings-v-delayed
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: ratings
            version: v-delayed
        spec:
          containers:
          - name: ratings
            image: istio/examples-bookinfo-ratings-v-delayed:latest
            imagePullPolicy: IfNotPresent
            ports:
            - containerPort: 9080
    EOF
    {{< /text >}}

1.  Direct the traffic to the faulty `ratings`:

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: ratings
    spec:
      host: ratings
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
      subsets:
      - name: v-delayed
        labels:
          version: v-delayed

    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: ratings
    spec:
      hosts:
        - ratings
      http:
      - route:
        - destination:
            host: ratings
            subset: v-delayed
    EOF
    {{< /text >}}

1.  Check your Kiali console,
    [http://my-kiali.io/kiali/console](http://my-kiali.io/kiali/console), the graph of your namespace.

    Note that now `productpage` turned orange while `reviews` and `ratings` turned red. Notice the red edges and the
    error rate of the HTTP traffic on the right.

    {{< image width="80%"
        link="images/kiali-delayed-ratings.png"
        caption="Kiali Graph Tab with delayed ratings"
        >}}

1.  Approximately half of the times you will get a delay of about 3 seconds and finally an error about
    `ratings` being unavailable. Note that in this case you both caused a delay for your user and also failed at least
    to display the reviews (without ratings). You have a cascading failure, that is, the fault propagated from `ratings`
    through `reviews` to `productpage`. Not good.

    {{< image width="80%"
        link="images/bookinfo-reviews-unavailable.png"
        caption="Bookinfo application: reviews unavailable"
        >}}

1.  Mitigate the problem with the help of Istio. Define a timeout on the call from `reviews` to `ratings`.
    Let `reviews` wait for 0.8 seconds and then return the reviews without ratings to `productpage`. This way
    `productpage` will not be delayed and will be able to display at least the reviews correctly (without ratings).

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: ratings
    spec:
      hosts:
        - ratings
      http:
      - route:
        - destination:
            host: ratings
            subset: v-delayed
        timeout: 0.8s
    EOF
    {{< /text >}}

1.  Access your application's webpage several times. Now you do not receive _reviews unavailable_ any more and you
    do not have three seconds delay! Only the `ratings` is unavailable about two thirds of the times. You managed to
    localize the problem, the failing `ratings` microservice does not influence the display of `reviews`. You provide
    partial functionality to the user in the face of occurring failures and do not leave your user waiting for the
    results of the whole webpage.

    {{< image width="80%"
        link="images/bookinfo-ratings-unavailable.png"
        caption="Bookinfo application: ratings unavailable"
        >}}

1.  Check your Kiali console,
    [http://my-kiali.io/kiali/console](http://my-kiali.io/kiali/console), the graph of your namespace.

    Note that now `productpage` turned green and the error rate of the HTTP traffic is reduced.

    {{< image width="80%"
        link="images/kiali-timeout-to-ratings.png"
        caption="Kiali Graph Tab with faulty ratings"
        >}}

1.  Remove the _v-delayed_ version of `ratings` and recreate the destination rule and the virtual service to route to
    _ratings v1_:

    {{< text bash >}}
    $ kubectl apply -f {{< github_file >}}/samples/bookinfo/networking/destination-rule-all-mtls.yaml
    $ kubectl delete virtualservice ratings
    $ kubectl delete deployment ratings-v-delayed
    {{< /text >}}