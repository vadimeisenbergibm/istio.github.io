1.  Determine the ingress IP and port for `cluster2`.

    1.   Set the current context of `kubectl` to `CTX_CLUSTER2`

        {{< text bash >}}
        $ export ORIGINAL_CONTEXT=$(kubectl config current-context)
        $ kubectl config use-context $CTX_CLUSTER2
        {{< /text >}}

    1.   Follow the instructions in
        [Determining the ingress IP and ports](/docs/tasks/traffic-management/ingress/#determining-the-ingress-ip-and-ports),
        to set the `INGRESS_HOST` and `SECURE_INGRESS_PORT` environment variables.

    1.  Restore the previous `kubectl` context:

        {{< text bash >}}
        $ kubectl config use-context $ORIGINAL_CONTEXT
        $ unset ORIGINAL_CONTEXT
        {{< /text >}}
