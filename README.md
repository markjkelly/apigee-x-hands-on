# apigee-x-hands-on

This document steps through the process of generating, deploying and testing a mock proxy on Apigee X, based on an Open API specification.

## Prerequisites
- An Apigee X organisation configured for external exposure. See [here](https://github.com/apigee/devrel/tree/main/tools/apigee-x-trial-provision) for details on provisioning an evaluation organisation.
- [NodeJS](https://nodejs.org/en/) LTS version or above.
- [Apigee Sackmesser](https://github.com/apigee/devrel/tree/main/tools/apigee-sackmesser)
- [apigeecli](https://github.com/srinandan/apigeecli)
- [git](https://git-scm.com/)
- [jq](https://stedolan.github.io/jq/)

## Generate, Deploy and Test a Mock Proxy

1. First you will need to set up some environment variables.
```sh
export TOKEN=$(gcloud auth application-default print-access-token)
export APIGEE_ENV=<apigee-environment-name>
export APIGEE_ORG=<gcp-project-name>
```

2. Clone the Apigee DevRel repo and install the dependencies for the oas-apigee-mock tool. 
```
git clone https://github.com/apigee/devrel.git
cd devrel/tools/oas-apigee-mock/
npm install
```

3. Generate a mock proxy from the `orders-apikey-header.yaml` Open API specification.
```
node bin/oas-apigee-mock generateApi web-orders-proxy-v1 -s test/oas/orders-apikey-header.yaml -o
```

4. Add a base configuration file to the generated bundle. This step will not be required once issue [338](https://github.com/apigee/devrel/issues/338) is resolved.
```
echo '<APIProxy name="web-orders-proxy-v1"/>' > api_bundles/web-orders-proxy/apiproxy/web-orders-proxy-v1.xml
```

5. Update the `before(function ()` block in `test/features/step_definitions/init.js` with your organisation's hostname. This will be your `RUNTIME_HOST_ALIAS` if you followed the [Apigee X Trial Provisioning](https://github.com/apigee/devrel/tree/main/tools/apigee-x-trial-provision) script.
```
before(function () {
  this.apickli = new apickli.Apickli(
    "https",$RUNTIME_HOST_ALIAS
  );
```

6. Update the test suite with your proxy name.

7. Run the test suite and verify it fails.
```
./node_modules/.bin/cucumber-js test/features/orders-apikey-header.feature --format json:test/test_report.json --publish-quiet
```

8. Deploy your mock proxy to your Organisation.
```
sackmesser deploy -d "$PWD/api_bundles/web-orders-proxy-v1" --googleapi -t "$TOKEN" -o "$APIGEE_ORG" -e "$APIGEE_ENV"
```

9. Turn on trace, run test suite and verify it fails.
```
./node_modules/.bin/cucumber-js test/features/orders-apikey-header.feature --format json:test/test_report.json --publish-quiet
```

10. Create a developer, app and product to test the proxy.
```
apigeecli products create -t "$TOKEN" -o "$APIGEE_ORG" --name "web-orders" --proxies "web-orders-proxy-v1" --envs "$APIGEE_ENV" --approval "auto"
apigeecli devs create -t "$TOKEN" -o "$APIGEE_ORG" --email "web-orders@example.com" --user "web-orders@example.com" --first "Web" --last "Developer"
apigeecli apps create -t "$TOKEN" -o "$APIGEE_ORG" --email "web-orders@example.com" --prods "web-orders" --name "web-orders-app" > app.json

APIKEY=$(jq '.credentials[0].consumerKey' -r < app.json )
export APIKEY
echo "APIKEY is $APIKEY"
```

11. Turn on trace, run test suite and verify it now passes.
```
./node_modules/.bin/cucumber-js test/features/orders-apikey-header.feature --format json:test/test_report.json --publish-quiet
```
