# Continuous Delivery

[Continuous Delivery](https://en.wikipedia.org/wiki/Continuous_delivery) is a practice of performing frequent, small deployments which aims to reduce time, risk, and cost associated with producing and deploying new feature. Despite the benefits, this practice has its own set of challenges, one of them often being the technical complexity associated with setting up the pipeline to enable continuous, automated deployment.

ConductR now provides Continuous Delivery feature which aims to help your team reap the associated benefits of Continous Delivery, and at the same time reducing the technical complexity normally associated with this practice.

Continuous Delivery feature is delivered as a bundle hosted in Bintray. It accepts notification of new version of bundles to be deployed through the use of Bintray webhook. Once a webhook is received, the new version of the bundle will be deployed to replace an existing, running bundle. The new bundle will be scaled up in the lock-step fashion as the old bundle is scaled down. The deployment is complete when the old bundle has stopped and the new bundle is running with the same scale as the previously running old bundle.

If the existing bundle is deployed with a bundle configuration, this configuration will be preserved and applied to the new bundle automatically. This minimises the exposure of potentially sensitive data in the configuration throughout the deployment process.

Installation of the Continuous Delivery bundle itself is relatively straightforward, requiring a working ConductR cluster with ConductR HAProxy and a Bintray webhook.

Your team will be in the best position to utilise the Continuous Delivery feature by enabling [bintray publishing](BintrayPublishing) of successfully built bundle as part of Continuous Integration build.


## Example pipeline

Here's an example of Continuous Delivery pipeline where bundles are propagated into production.

Let there be 2 clusters: a staging cluster and a production cluster. Both cluster has Continuous Delivery bundle installed. The Continuous Delivery service on the staging environment is accessible from the outside world.

Let's assume the development team is using `git`, and all the code that has been promoted into `master` branch can be released once it has passed the test.

A CI server has been configured to publish bundle artefacts to Bintray upon successful build of recently merged commit into the `master` branch.

The Bintray webhook has been configured to target the staging environment. This would mean each successful merged commit into the `master` branch will be available for manual test on the staging environment.

The manual tests will be performed on the staging environment, and a particular version will be selected for production deployment. As part of the version selection process, older versions which hasn't been selected will be removed using the `conduct unload` command.

Once a particular version has been selected for a production deployment, an operator can use `conduct deploy` command to promote this particular version into the production environment. Once the deployed version is confirmed to be working, the older bundle is then unloaded from the production environment.

ConductR's Continuous Delivery feature doesn't prescribe how the deployment pipeline should look like. If the Bintray webhook in the example pipeline above is pointed to production cluster instead, it would mean continuous deployment to production instead.


## Requirements

From the perspective of Continuous Delivery setup, the operator is responsible for installation of the Continuous Delivery service of ConductR as well as configuration of Bintray webhook which communicates with Continuous Delivery service.

Continuous Delivery functionality requires a working ConductR Cluster with ConductR HAProxy deployed. ConductR HAProxy is required to allow Bintray webhook access to Continuous Delivery service. ConductR HAProxy installation instructions is available on the [[Install|Install]] page.

Continuous Delivery functionality requires Bintray webhook to be setup. As such, Bintray credentials with publish permission and package read/write entitlement is required.

## Bintray webhook secret

The webhook secret is the Bintray API key of the credentials owning the Bintray webhook.

For the purpose of this documentation, we will be using `bt-secret` in place of the actual API key value.

## Installing Continuous Delivery service

The Continous Delivery service needs to be configured with the webhook secret. Prepare the bundle configuration as such, replacing `bt-secret` with the actual Bintray API key value.

```
$ mkdir /tmp/cd-config
$ echo 'export BINTRAY_WEBHOOK_SECRET=bt-secret' > /tmp/cd-config/runtime.config.sh
$ shazar /tmp/cd-config
```

The bundle configuration for the Bintray webhook secret will be generated by running the commands above.

Load the Continuous Delivery bundle with the generated bundle configuration and run the service.

```
conduct load continuous-delivery $(find /tmp/cd-config-*.zip | head -n 1)
conduct run continuous-delivery --scale 2
```
The Continuous Delivery service will form a cluster among its instances, and the deployer which is responsible for deployment is sharded by bundle name and its compatible version.

Once the Continuous Delivery service has been started, it will be exposed via the proxy on port `9000` under the path `/deployments`. Ensure external access is available to this port and path to allow Continuous Delivery service to be invoked by Bintray webook.

## Bintray webhook setup

Execute the following command to create Bintray webhook.

```
curl -v \
     -u ${BINTRAY_USERNAME}:${BINTRAY_API_KEY} \
     -X POST \
     -H "Content-Type: application/json" \
     -d '{"url": "${CALLBACK_URL}", "method": "post"}' \
     "https://api.bintray.com/webhooks/${BINTRAY_SUBJECT}/${BINTRAY_REPO}/${BINTRAY_PACKAGE_NAME}"
```

|Term|Description|Example|
|----|-----------|-------|
|`BINTRAY_USERNAME`|The Bintray username. This can be obtained by signing into Bintray and viewing the user profile.|&nbsp;|
|`BINTRAY_API_KEY`|The API key of the user. This can be obtained by signing into Bintray and viewing the API key within the user profile.|`bt-secret` is used for our example.|
|`CALLBACK_URL`|The callback URL which will be invoked by the webhook.|&nbsp;|
|`BINTRAY_SUBJECT`|The Bintray subject who owns the `BINTRAY_REPO` in question.|`typesafe` is an example of Bintray subject which is accessible on [https://bintray.com/typesafe](https://bintray.com/typesafe).|
|`BINTRAY_REPO`|The Bintray repository where the bundle to be deployed resides.|`bundle` is an example of Bintray repo which is accessible on [https://bintray.com/typesafe/bundle](https://bintray.com/typesafe/bundle).<br>The repo owns various bundles such as `cassandra` which is accessible from [https://bintray.com/typesafe/bundle/cassandra](https://bintray.com/typesafe/bundle/cassandra).|
|`BINTRAY_PACKAGE_NAME`|The Bintray package of the bundle to be deployed.|`cassandra` is an example of Bintray package which is accessible on [https://bintray.com/typesafe/bundle/cassandra](https://bintray.com/typesafe/bundle/cassandra).|


Format the `CALLBACK_URL` as such.

```
${CD_BASE_URL}/deployments/${BINTRAY_SUBJECT}/${BINTRAY_REPO}/<subject>
```

|Term|Description|Example|
|----|-----------|-------|
|`CD_BASE_URL`|The URL where Continuous Delivery is exposed. This will normally be a load balancer which is mapped to proxy on port `9000` under the path `/deployments`.|`http://staging.acme.com`|
|`BINTRAY_SUBJECT`|Must match the `BINTRAY_SUBJECT` posted to `api.bintray.com`|&nbsp;|
|`BINTRAY_REPO`|Must match the `BINTRAY_REPO` posted to `api.bintray.com`|&nbsp;|

Once the setup is complete, the webhook can be tested using the following command. In the following example `0.0.1` is the test version number of the bundle in question.

```
curl -v \
     -u ${BINTRAY_USERNAME}:${BINTRAY_PASSWORD} \
     -X POST \
     "https://api.bintray.com/webhooks/${BINTRAY_SUBJECT}/${BINTRAY_REPO}/${BINTRAY_PACKAGE_NAME}/0.0.1"
```

Further details on the bintray webhook setup instructions can be found on the [https://bintray.com/docs/api/#_register_a_webhook](https://bintray.com/docs/api/#_register_a_webhook).

## Manual deployment

The `conduct deploy` command can be used to deploy a bundle manually through the Continuous Delivery pipeline, and particularly useful for rolling backward or rolling forward through deployments.

Configure the Continuous Delivery settings in the `~/.conductr/settings.conf` as such, replacing `bt-secret` with the actual Bintray API key value.

```
conductr {
  continuous-delivery {
    bintray-webhook-secret = bt-secret
  }
}
```

Invoke the command as such, where `<bundle-id>` is the [shorthand expression](DeployingBundlesOps#Bundle-shorthand-expression) of the bundle to be deployed.

```
conduct deploy <bundle-id>
```

For bundles deployed with bundle configuration, note that `conduct deploy` command will apply configuration from previous version of the running bundle to the version of the bundle presently being deployed.

## Housekeeping

After a successful deployment, Continuous Delivery service will not unload the older bundle which has been stopped to allow the operator to quickly fallback and run the older bundle.

However this would mean the older bundle will be kept within ConductR until the older bundle is unloaded. As such, it is a good practice to unload older versions of bundles on a regular basis.

## Compatibility version

Continuous Delivery service will only deploy versions which are binary compatible.

The `compatibilityVersion` bundle configuration will be used to check if the versions being deployed is binary compatible with the version being replaced. ConductR expects that two different versions of a bundle having the same `compatibilityVersion` will be binary compatible.

Refer to [Bundle configuration](BundleConfiguration) for more details on `compatibilityVersion`.

## Configuration changes

Continuous Delivery service will reapply configuration from existing running bundle to the bundle being deployed.

If a new configuration needs to be deployed, manual deployment will be required.

