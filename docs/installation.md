# Installation

## 1. Set up `bentoctl`

See the following guide: [`bentoctl`: the Bento deployment command line management tool](./bentoctl.md)

Before running any installation steps, make sure you've set up `bentoctl` and have activated its
virtual environment.

## 2. Provision configuration files

### Instance-specific environment variable file: `local.env`

Depending on your use, development or deployment, you will need to copy the right template file
to `local.env` in the root of the `bento` folder, i.e., the repository folder:

```bash
# Dev
cp ./etc/bento_dev.env local.env

# Deployment
cp ./etc/bento_deploy.env local.env
```

Then, modify the values as needed; depending on if you're using the instance for development or deployment.

#### Development example

For an example of a semi-completed development configuration, see `etc/bento_dev.env`.
In the step above, this file was copied to `local.env`, and **you must now edit `local.env` to specify secrets and other
deployment-specific values.**

You should at least fill to the following settings in dev mode (it may differ for a production setup), which are not set
in the example file:

- `BENTOV2_SESSION_SECRET`
- `BENTO_AUTH_DB_PASSWORD`
- `BENTO_AUTHZ_DB_PASSWORD`
- `BENTO_AGGREGATION_CLIENT_SECRET`
- `BENTO_WES_CLIENT_SECRET`

If the internal OIDC identity provider (IdP) is being used (by default, Keycloak), variables specifying default
credentials should also be provided. The _admin_ credentials are used to connect to the Keycloak UI for authentication
management (adding users, getting client credentials, ...). The _test_ credentials will be used to authenticate on the
Bento Portal.

```bash
BENTOV2_AUTH_ADMIN_USER=testadmin
BENTOV2_AUTH_ADMIN_PASSWORD=testpassword123

BENTOV2_AUTH_TEST_USER=testuser
BENTOV2_AUTH_TEST_PASSWORD=testpassword123
```

If using an _external_ identity provider, adjust the following auth variables according to the external IdP's
specifications:

```bash
BENTOV2_AUTH_CLIENT_ID=local_bentov2
BENTOV2_AUTH_REALM=bentov2

BENTOV2_AUTH_WELLKNOWN_PATH=/auth/realms/${BENTOV2_AUTH_REALM}/.well-known/openid-configuration
```

If an _external_ Keycloak instance is being used and you would like to set it up using `./bentoctl.bash init-auth`, follow the steps above for external IdPs and then below:

- Set up a keycloak instance and provide its URL in `BENTOV2_AUTH_PUBLIC_URL`
- Make the desired Bento realm and update `BENTOV2_AUTH_REALM` accordingly
- Create an administrative user account in the Bento realm
- In _Realm Roles_, create an admin role, which we will give to the administrative user later
- In the admin role, go to the associated roles tab and assign all the **realm-management** roles to the admin realm role.
- In the admin user page, select the _Role Mapping_ tab and click on the _Assign Role_ button
- Select _Filter By Realm Roles_, select the admin role and click _Assign_
- The user account can then be used to remotely configure the realm with the admin API
- Set the credential variables in `local.env`
  - To provide credentials during `init-auth` (**recommended**)
    - `BENTOV2_AUTH_ADMIN_USER=''` & `BENTOV2_AUTH_ADMIN_PASSWORD=''`
    - You will then be prompted to provide the credentials
  - To store the credentials in an environment variable
    - `BENTOV2_AUTH_ADMIN_USER=<admin username>` & `BENTOV2_AUTH_ADMIN_PASSWORD=<admin password>`
- Set `BENTOV2_USE_EXTERNAL_KEYCLOAK=1` in `local.env`
- Run `./bentoctl.bash init-auth`

> This administrative user account should only be used to configure a Keycloak realm for a Bento instance.
> As such, once your Bento instance is properly configured with your remote Keycloak, we recommend that you unassign the admin realm role from the admin user and reset credentials. Doing so mitigates the potential security risks caused by leaked admin credentials.
>
> If you need to re-run `bentoctl init-auth` later, you can set new credentials and reassign the admin role for the configuration process.
>
> `bentoctl init-auth` would skip test user creation. Test users can be created using the admin console for the external keycloak.

### Bento Public configuration

Then, copy the `bento_public` configuration file to its correct location for use by Katsu,
Bento's clinical/phenotypic metadata service:

```bash
# public service configuration file. Required if BENTOV2_USE_BENTO_PUBLIC flag is set to `1`
# See Katsu documentation for more information about the specifications
./bentoctl.bash init-config katsu
```

### Beacon configuration

If using Beacon, first copy the configuration file:

```bash
./bentoctl.bash init-config beacon
```

Then update any config values as needed at `lib/beacon/config/beacon_config.json`
and `lib/beacon/config/beacon_cohort.json`.

If using the Beacon network, copy the configuration file:

```bash
./bentoctl.bash init-config beacon-network
```

and update values at `lib/beacon/config/beacon_network_config.json`.

### Gohan configuration

#### Production

When deploying a Bento node that needs to serve variants data, it is recommended to allot larger portions of your
compute resources to both `bentov2-gohan-api` and `bentov2-gohan-elasticsearch`.
Since variants are stored in memory in Elasticsearch, set the `BENTOV2_GOHAN_ES_MEM_LIM` variable to an
appropriate value.

Alloting additional CPUs with `BENTOV2_GOHAN_API_CPUS` and `BENTOV2_GOHAN_ES_CPUS` will result in shorter ingestion
times. Consider this option if you need to ingest a large number of VCFs.

Go through the official Elasticsearch
[checklist](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-prod-prerequisites)
for production deployments on Docker.

The following configurations **MUST** be applied to the host machine deploying the Elasticsearch container:

- [Set `vm.max_map_count`](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_set_vm_max_map_count_to_at_least_262144)
- [Disable swapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-configuration-memory.html#swappiness)

#### Elasticsearch JVM options

In **production**, using the default JVM options provided by Elasticsearch is **recommended**.

In **development**, modifying the JVM heap size could be needed if Elasticsearch's memory limit
is low and you are trying to ingest VCFs.
The heap size can be modified by providing a config file to the `bentov2_gohan-elasticsearch` container:

```bash
cp ./etc/default.gohan.es.jvm.options ./lib/gohan/es_jvm_options/jvm.options
```

Set `Xms` and `Xmx` to no more than 50% of the `BENTOV2_GOHAN_ES_MEM_LIM` value.
For more details, check the official Elasticsearch doc on
[heap size](https://www.elastic.co/guide/en/elasticsearch/reference/current/advanced-configuration.html#set-jvm-heap-size).

## 3. _Development only:_ create self-signed TLS certificates

First, set up your local Bento and Keycloak hostnames (something like `bentov2.local`, `portal.bentov2.local`, and
`bentov2auth.local`) in the `.env` file. You can then create the corresponding TLS certificates for local development.

Setting up the certificates with `bentoctl` can be done in a single command.
From the project root, run

```bash
./bentoctl.bash init-certs
```

> **NOTE:** This command will skip all certificate generation if it detects previously created files.
> To force an override, simply add the option `--force` / `-f`.

After creating the three certificates, it is worth ensuring your browser has security exceptions for these
certificates and domains. Navigate to each of the three domains mentioned above and add security exceptions
to ensure cross-origin requests will occur correctly.

## 4. _Development only:_ Hosts file configuration

Ensure that the local domain names are set in the machines `hosts` file (for Linux users, this is likely
`/etc/hosts`, and in Windows, `C:\Windows\System32\drivers\etc\hosts`) pointing to either `localhost`, `127.0.0.1`,
or `0.0.0.0`, depending on whichever gets the job done on your system.

With the default development configuration, this might look something like:

```
# ... system stuff above
127.0.0.1	bentov2.local
127.0.0.1	portal.bentov2.local
127.0.0.1	bentov2auth.local
# ... other stuff below
```

If you are working with cBioPortal, you will need another line:

```
# ...
127.0.0.1   cbioportal.bentov2.local
# ...
```

Editing `/etc/hosts` is **not needed** in production, since the domains should have DNS records.

Make sure these values match the values in the `.env` file and what was issued in the self-signed certificates, as
specified in the step above.

## 5. Initialize and boot the gateway

> NOTE: `./bentoctl.bash` commands seen here aren't the only tools for operating this cluster.
> Run `./bentoctl.bash --help` for further documentation.

```bash
# Once the certificates are ready, initialize various aspects of the cluster:
./bentoctl.bash init-all
# Which is equivalent to:

#   # Once the certificates are ready, initialize the cluster configs secrets
#   ./bentoctl.bash init-dirs
#   ./bentoctl.bash init-docker
#   ./bentoctl.bash init-secrets
#
#   # Initialize bento_web and bento_public
#   ./bentoctl.bash init-web private
#   ./bentoctl.bash init-web public

# If you are running the bentoV2 with the use of an internal identity provider (defaults to Keycloak),
# i.e setting BENTOV2_USE_EXTERNAL_IDP=0, run both
./bentoctl.bash run auth
./bentoctl.bash run gateway
# and
./bentoctl.bash init-auth
```

After running `init-auth`, be sure to put all client secrets into your `local.env` file!

**If using an external identity provider**, only start the cluster's gateway after setting various `*_CLIENT_SECRET`
variables in your local environment file:

```bash
./bentoctl.bash run gateway
```

### Note on Keycloak

This last step boots and configures the local OIDC provider (**Keycloak**) container and reconfigures the gateway to
utilize new variables generated during the OIDC configuration.

> NOTE: by default, the `gateway` service _does_ need to be running for this to work as the configuration will pass via
> the URL set in the `.env` file which points to the gateway.
>
> If you do not plan to use the built-in OIDC provider, you will have to handle auth configuration manually.

## 6. Configure permissions

### a. Create superuser permissions in the Bento authorization service

First, run the authorization service and then open a shell into the container:

```bash
./bentoctl.bash run authz
./bentoctl.bash shell authz
```

Then, run the following command for each user ID you wish to assign superuser permissions to:

```bash
bento_authz assign-all-to-user iss sub
```

Where `iss` is the issuer (for example, `https://bentov2auth.local/realms/bentov2`) and `sub` is the user (subject) ID,
which in Keycloak should be a UUID.

### b. Create grants for the Workflow Execution Service (WES) OAuth2 client

Run the following commands to set up authorization for the WES client.
**Don't forget to replace `<ISSUER_HERE>` with the issuer URL!**

```bash
# This grant is a temporary hack to get permissions working for v12/v13. In the future, it should be removed.
bento_authz create grant \
  '{"iss": "<ISSUER_HERE>", "client": "wes"}' \
  '{"everything": true}' \
  'view:private_portal'

# This grant gives permission to access and ingest data into all projects and the reference genome service
bento_authz create grant \
  '{"iss": "<ISSUER_HERE>", "client": "wes"}' \
  '{"everything": true}' \
  'query:data' 'ingest:data' 'ingest:reference_material' 'delete:reference_material'
```

### c. Create a grant for the aggregation and Beacon services

Run the following commands to set up authorization for the aggregation/Beacon client.
**Don't forget to replace `<ISSUER_HERE>` with the issuer URL!**

```bash
# In the future, view:private_portal will need to be removed from this grant.
bento_authz create grant \
  '{"iss": "<ISSUER_HERE>", "client": "aggregation"}' \
  '{"everything": true}' \
  'query:data' 'view:private_portal'
```

### d. Configure public data access for all users, including anonymous visitors (if desired):

To configure public data access, run the following command in the authorization service container. Note that with the
`full` value, **THIS GIVES FULL DATA ACCESS TO EVERYONE WHO VISITS YOUR INSTANCE!**

```bash
# Configure public data access
# ----------------------------
# The level below ("counts") preserves previous functionality. Other possible options are:
#  - none - will do nothing.
#  - bool - for censored true/false discovery, but in effect right now forbids access.
#  - counts - for censored count discovery.
#  - full - allows full data access (record-level, including sensitive data such as IDs), uncensored counts, etc.
bento_authz public-data-access counts
```

### e. Assign portal access to all users in the instance realm

We added a special permission, `view:private_portal`, to Bento v12/v13 in order to carry forward the current
'legacy' authorization behaviour for one more major version. This permission currently behaves as a super-permission,
allowing all actions within the private portal. **However,** in the future, this permission will do almost _nothing._

To carry forward legacy behaviour of all users in the instance realm being able to do everything, run the following
command in the authorization service container:

```bash
# Create the grant
bento_authz create grant \
  '{"iss": "ISSUER_HERE", "client": "WEB_CLIENT_ID_HERE"}' \
  '{"everything": true}' \
  'view:private_portal'
```

Where `WEB_CLIENT_ID_HERE` is the OAuth2 client the web portal uses, i.e., the
value in the `BENTOV2_AUTH_CLIENT_ID` environment variable. On local instances,
this is set to `local_bentov2` by default.

## 7. _Production only:_ set up translations and branding

Now that Bento Public and Web have been initialized by either `./bentoctl.bash init-all` or
`./bentoctl.bash init-web <public|web>`, translation files and branding (logos) can be configured as necessary.

**For branding (logos)**, copy files to the following paths:

- A logo which works on dark backgrounds should be placed at `lib/public/branding.png` and `lib/web/branding.png`.
- A logo which works on light backgrounds should be placed at `lib/public/branding.lightbg.png`. This is primarily
  useful for Beacon Network.

**For translations** (which apply only to Bento Public), adjust the default translation set as necessary:

```js
// lib/public/translations/<en|fr>.json

{
  "Age": "Age",
  "Sex": "Sex",
  "Verbal consent date": "Verbal Consent Date",
  "Functional status": "Functional Status",
  "Lab Test Result": "Lab Test Results",
  "Experiment Types": "Experiment Types",
  "Demographics": "Demographics",
  "MALE": "MALE",
  "FEMALE": "FEMALE",
  "I have no problems in walking about": "I have no problems in walking about",
  "Results": "Results"
}


{
  "MALE": "HOMME",
  "FEMALE": "FEMME",
  "Age": "Âge",
  "Sex": "Sexe",
  "Demographics": "Démographie",
  "Verbal consent date": "Date de consentement verbal",
  "Functional status": "Statut fonctionnel",
  "Lab Test Result": "Résultats des tests de laboratoire",
  "Experiment Types": "Types d'expériences",
  "I have no problems in walking about": "Je n’ai aucun problème à marcher",
  "Results": "Résultats"
}
```

## 8. Start the cluster

```bash
./bentoctl.bash run all
# or
./bentoctl.bash run
# (these are synonymous)
```

to run all Bento services.

### Stopping and cleaning the cluster

Run

```bash
./bentoctl.bash stop all
```

to shut down the whole cluster.

To remove the Docker containers, run the following:

```bash
./bentoctl.bash clean all
```

> NOTE: application data does persist after cleaning
> (depending on data path, e.g., `./data/[auth, drs, katsu]/...` directories)

## 9. Ingest reference material

To support genomic data and gene querying, a reference genome (FASTA) and GFF3 gene catalog must be ingested.
Follow these steps:

1. Download your reference genome and GFF3 gene catalog files from RefSeq or a similar database.
2. Upload these files to your Bento instance's drop box.
3. Have a Phenopackets JSON-style ontology term (from the
   [`NCBITaxon` ontology](https://github.com/obophenotype/ncbitaxon)) ready, e.g.:
   ```json
   { "id": "NCBITaxon:9606", "label": "Homo sapiens" }
   ```
4. Navigate to `/data/manager/ingestion` in the Bento private portal and ingest the reference FASTA and GFF3 into the
   reference service. This will take some time.

### Set Up Gohan's Gene Catalogue (_Legacy instructions_)

To enable gene querying support in versions of Bento before v16, follow these steps to set up Gohan's gene catalogue:

1. **Access the Services Portal**:

   - Navigate to the `Services` tab on the portal.

2. **Initiate Gohan Request**:

   - Click the `Make Request` button for Gohan.

3. **Edit and Trigger Ingestion Endpoint**:

   - Modify the endpoint to `genes/ingestion/run`.
   - Click `Get` to initiate Gohan's download and processing of the default GenCode `.gtk` files from the internet.

4. **Monitor the Ingestion Process**:

   - Use the endpoint `genes/ingestion/requests` to track the progress of the ingestion process.

5. **Access the Gene Catalogue**:
   - Once the ingestion process is complete, the gene catalogue will be available at `genes/overview`.
