# ConductR release notes

## 2.1

This release broadly consists of the following functionality:

* [Open Container Initiative (OCI) image support](#OCI_Image_Support);
* [a licensing mechanism to enable free usage](#Free_Licensing); and
* [an experimental autoscaler](#An_Experimental_Autoscaler) (!).

### OCI Image Support

We are proud to announce ConductR as being the world's first cluster manager to support the OCI image format. OCI image support equates to running Docker images directly within ConductR. The key benefit to the you is the avoidance of “vendor lock-in”. The benefits of OCI images are described well here: https://www.opencontainers.org/faq#faq5.

### Free Licensing

The new licensing mechanism permits a single ConductR Agent to be used absolutely free in production and requiring no registration. If you register with us then you will receive a license and then be able to use up to 3 ConductR Agents in production. If you are a customer then life will continue as normal.

Please visit the [migration guide](MigrationGuide#Production_Suite_Licensing) for additional instructions.

### An Experimental Autoscaler

The experimental auto-scaler involves having instrumented HAProxy via conductr-haproxy, our bridge between ConductR and HAProxy. HAProxy is interesting because it provides data in relation to what a business SLA would require e.g. "95th percentile response times for the account service should not exceed 500ms”. We’ve bundled up an amazing off-the-shelf free tool named Riemann (http://riemann.io/). Riemann is essentially a stream processor - you send it streams of data and then, using configuration declared in Clojure (a functional programming language), you can pretty much do anything. We've provided some configuration that will measure the 95th percentile response time of a test-bundle and scale it up if it detects latency in performance.

For more information, including how to extend and configure it, please visit the [documentation on autoscaling](Autoscaler).