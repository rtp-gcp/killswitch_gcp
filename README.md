# killswitch_gcp
killswitch for gcp

# credits
Used [this](https://github.com/aioverlords/Google-Cloud-Platform-Killswitch) repo as a reference



# Billing 
1. Got to budgetst & Alerts
2. Create monthly budget for time range
3. Specify all projects and all services
4. Specify amount->Budget type as specified amount
5. Specify Target amount as $20
6. Specify Actions 100% of budget, trigger on actual
7. Enable Email alerts to billing admins and users
8. Connect a Pub/Sub topic to this budget
9. Specify a Cloud Pub/Sub topic - create one and give it a name

# Cloud Function
1. create function to trigger on the Cloud Pub/Sub topic above
2. Specify retry on failure
3. Change Entry point to stopBilling
4. Add the following code NOTE: change PROJECT_ID to proper value
```
const {
    google
} = require('googleapis');
const {
    GoogleAuth
} = require('google-auth-library');

const PROJECT_ID = 'macgyver-services-production';
const PROJECT_NAME = `projects/${PROJECT_ID}`;
const billing = google.cloudbilling('v1').projects;

exports.stopBilling = async pubsubEvent => {

    console.log(pubsubEvent.data);

    const pubsubData = JSON.parse(
        Buffer.from(pubsubEvent.data, 'base64').toString()
    );

    console.log(pubsubData);
    console.log(pubsubData.costAmount);
    console.log(pubsubData.budgetAmount);

    if (pubsubData.costAmount <= pubsubData.budgetAmount) {
        console.log("No action necessary.");
        return `No action necessary. (Current cost: ${pubsubData.costAmount})`;
    }

    if (!PROJECT_ID) {
        console.log("no project specified");
        return 'No project specified';
    }

    _setAuthCredential();

    const billingEnabled = await _isBillingEnabled(PROJECT_NAME);
    if (billingEnabled) {
        console.log("disabling billing");
        return _disableBillingForProject(PROJECT_NAME);
    } else {
        console.log("billing already disabled");
        return 'Billing already disabled';
    }
};

/**
 * @return {Promise} Credentials set globally
 */
const _setAuthCredential = () => {
    const client = new GoogleAuth({
        scopes: [
            'https://www.googleapis.com/auth/cloud-billing',
            'https://www.googleapis.com/auth/cloud-platform',
        ],
    });

    // Set credential globally for all requests
    google.options({
        auth: client,
    });
};

/**
 * Determine whether billing is enabled for a project
 * @param {string} projectName Name of project to check if billing is enabled
 * @return {bool} Whether project has billing enabled or not
 */
const _isBillingEnabled = async projectName => {
    try {
        const res = await billing.getBillingInfo({
            name: projectName
        });
        console.log(res);
        return res.data.billingEnabled;
    } catch (e) {
        console.log(
            'Unable to determine if billing is enabled on specified project, assuming billing is enabled'
        );
        return true;
    }
};

/**
 * Disable billing for a project by removing its billing account
 * @param {string} projectName Name of project disable billing on
 * @return {string} Text containing response from disabling billing
 */
const _disableBillingForProject = async projectName => {
    const res = await billing.updateBillingInfo({
        name: projectName,
        resource: {
            billingAccountName: ''
        }, // Disable billing
    });
    console.log(res);
    console.log("Billing Disabled");
    return `Billing disabled: ${JSON.stringify(res.data)}`;
};
```
5. changed package.json to this
```
{
 "name": "cloud-functions-billing",
 "version": "0.0.1",
 "dependencies": {
   "google-auth-library": "^2.0.0",
   "googleapis": "^52.0.0"
 }
}
``

# IAM and Admin
1. Service Accounts in left side panel - create killswitch-sa service account`
2. IAM in left side panel - Add one role - Project Billing Manager

# Cloud Function
1. Go back to killswitch cloud function
2. Click Edit
3. Click Runtime, build, connections, and security settings
4. Modify "Runtime service account" to use killswitch service account



# Testing
1. Go to topics
2. Click the topic
3. at bottom, three tabs, "Subscriptions, snapshots, messages", click messages
4. click publish message
5. Use this message body
```
{
    "budgetDisplayName": "name-of-budget",
    "alertThresholdExceeded": 1.0,
    "costAmount": 100.01,
    "costIntervalStart": "2022-01-09T00:00:00Z",
    "budgetAmount": 100.00,
    "budgetAmountType": "SPECIFIED_AMOUNT",
    "currencyCode": "USD"
}
```

Once you get testing working, it will act wonky.  You will not be able to see the cloud function source for example.
At this point, the web services associated with this service account will stop.  You will get emails yadda yadda.

Go back to Billing and then at bottom in left hand side, click account management to renable billing.
