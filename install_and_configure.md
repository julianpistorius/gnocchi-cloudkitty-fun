Title:          Gnocchi & CloudKitty Spike  
Author:         Julian Pistorius  
Date:           2016-05-12  
html header:    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.2.0/styles/default.min.css">  
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.2.0/highlight.min.js"></script>  
<script>hljs.initHighlightingOnLoad();</script>  
html header:    <meta http-equiv="refresh" content="2000">  

# Gnocchi & CloudKitty Spike

## TODO

- Gnocchi
    - [x] Install Gnocchi from source
    - [x] Configure and get Gnocchi running
    - [x] Install Gnocchi client from source
    - [x] Create resource & metrics
    - [x] Push measures to Gnocchi
    - [x] Get measures for a metric

- CloudKitty
    - [x] Install CloudKitty from source
    - [ ] Configure CloudKitty
        - [ ] Ask CloudKitty guys whether oslo.messaging is necessary
        - Or can I use 'fake'?
          <http://docs.openstack.org/developer/oslo.messaging/drivers.html>
        - [x] Fake seems to work so far. Just need to populate fake tenants CSV
            - CloudKitty tenant_id is Gnocchi project_id
            - There is a bug in CloudKitty:
                <https://review.openstack.org/#/c/314655/>
        - [x] Change upstream for CloudKitty to git.openstack
        - [x] Download Stéphane's patch for CloudKitty
        - [ ] Test patch
        - [ ] Possibly commit patch to that patch to address concerns
    - [ ] Create rating plan to approximate AU/SU scenario
    - [ ] Connect Gnocchi to CloudKitty
    - [ ] Send old instance status history to Gnocchi
    - [ ] See what shows up in CloudKitty
    - [ ] Failing all this, use CloudKitty directly against Ceilometer for
        now.
- [ ] Grafana


## Notes for setting Gnocchi up without OpenStack

### 1. Install from source. And run tests. 

See: <http://gnocchi.xyz/install.html>


```bash
git clone https://github.com/openstack/gnocchi.git
mkvirtualenv gnocchi
cd gnocchi
pip install -e .[file,postgresql,test]
./run-tests.sh
```

### 2. Create database

```bash
createdb gnocchi
psql postgres
CREATE ROLE gnocchi WITH ENCRYPTED PASSWORD 'mygnocchipasswordisawesome';
GRANT ALL PRIVILEGES ON DATABASE gnocchi TO gnocchi;
ALTER ROLE gnocchi WITH LOGIN;
\q
```

### 3. Generate and edit the config file

See: <http://gnocchi.xyz/configuration.html>

```bash
mkdir /usr/local/var/lib/gnocchi
oslo-config-generator --config-file=etc/gnocchi/gnocchi-config-generator.conf --output-file=etc/gnocchi/gnocchi.conf
cp etc/gnocchi/gnocchi.conf etc/gnocchi/gnocchi.conf.dist
# vim etc/gnocchi/gnocchi.conf
diff --unified=1 etc/gnocchi/gnocchi.conf.dist etc/gnocchi/gnocchi.conf
```

Output:

> ```diff
> --- etc/gnocchi/gnocchi.conf.dist	2016-05-12 12:22:02.000000000 -0700
> +++ etc/gnocchi/gnocchi.conf	2016-05-12 13:10:59.000000000 -0700
> @@ -10,2 +10,3 @@
>  #debug = false
> +debug = true
>  
> @@ -112,2 +113,3 @@
>  #port = 8041
> +port = 8041
>  
> @@ -115,2 +117,3 @@
>  #host = 0.0.0.0
> +host = 127.0.0.1
>  
> @@ -305,2 +308,3 @@
>  #url = <None>
> +url = postgresql://gnocchi:mygnocchipasswordisawesome@localhost/gnocchi
>  
> @@ -503,2 +507,3 @@
>  #host = 0.0.0.0
> +host = 127.0.0.1
>  
> @@ -539,3 +544,3 @@
>  # Storage driver to use (string value)
> -#driver = file
> +driver = file
>  
> @@ -565,2 +570,3 @@
>  #file_basepath = /var/lib/gnocchi
> +file_basepath = /usr/local/var/lib/gnocchi
> ```

### 4. Edit the api-paste.ini file

TODO? No changes yet.

```bash
cp etc/gnocchi/api-paste.ini etc/gnocchi/api-paste.ini.dist
# vim etc/gnocchi/api-paste.ini
```

```bash
diff --unified=1 etc/gnocchi/api-paste.ini.dist etc/gnocchi/api-paste.ini
```

Output:

> ```diff
> ```


### 5. Edit the policy.json file

TODO? No changes yet.

```bash
cp etc/gnocchi/policy.json etc/gnocchi/policy.json.dist
# vim /etc/gnocchi/policy.json
```

```bash
diff --unified=1 etc/gnocchi/policy.json.dist etc/gnocchi/policy.json
```

Output:

> ```diff
> ```


### 6. Initialize the Gnocchi indexer and storage

```bash
gnocchi-upgrade --config-dir=etc/gnocchi/
echo '\dt' | psql -U gnocchi gnocchi
l /usr/local/var/lib/gnocchi/
```

Output:

> ```
>                List of relations
>  Schema |        Name         | Type  |  Owner  
> --------+---------------------+-------+---------
>  public | alembic_version     | table | gnocchi
>  public | archive_policy      | table | gnocchi
>  public | archive_policy_rule | table | gnocchi
>  public | metric              | table | gnocchi
>  public | resource            | table | gnocchi
>  public | resource_history    | table | gnocchi
>  public | resource_type       | table | gnocchi
> (7 rows)
> 
> total 0
> drwxr-xr-x  4 julianp  admin  136 May 12 13:15 .
> drwxrwxr-x  4 julianp  admin  136 May 12 13:06 ..
> drwxr-xr-x  2 julianp  admin   68 May 12 13:15 measure
> drwxr-xr-x  2 julianp  admin   68 May 12 13:15 tmp
> ```

### 7. Run Gnocchi

See: <http://gnocchi.xyz/running.html>

```bash
gnocchi-api --config-dir=etc/gnocchi/
gnocchi-metricd --config-dir=etc/gnocchi/
```

Output:

> ```
> 2016-05-12 13:35:30.670 17319 INFO werkzeug [-]  * Running on http://127.0.0.1:8041/ (Press CTRL+C to quit)
> ```

TODO: Switch logging level to INFO. DEBUG is a bit noisy.


### 8. Check Gnocchi status

```bash
curl -H "x-roles: admin" http://127.0.0.1:8041/v1/status | python -mjson.tool
```

Output:

> ```json
> {
>     "storage": {
>         "measures_to_process": {},
>         "summary": {
>             "measures": 0,
>             "metrics": 0
>         }
>     }
> }
> ```


### 9. Install Gnocchi client

```bash
cd ..
git clone https://github.com/openstack/python-gnocchiclient.git
mkvirtualenv python-gnocchiclient
cd python-gnocchiclient/
pip install -e .
```

### 10. Set up environment for Gnocchi client

```bash
export OS_AUTH_PLUGIN=gnocchi-noauth
export GNOCCHI_ENDPOINT=http://127.0.0.1:8041
export GNOCCHI_USER_ID=99aae-4dc2-4fbc-b5b8-9688c470d9cc
export GNOCCHI_PROJECT_ID=c8d27445-48af-457c-8e0d-1de7103eae1f
# For Charles proxy:
export HTTP_PROXY=http://127.0.0.1:8888/
export HTTPS_PROXY=http://127.0.0.1:8888/
```

## Fun things with Gnocchi

### Create a new resource type

```bash
echo "SELECT * FROM resource_type\x" | psql -U gnocchi gnocchi  
```

Output:

> ```
> Expanded display is on.
> -[ RECORD 1 ]-----------------------------------
> name       | generic
> tablename  | generic
> attributes | {}
> ```

```bash
gnocchi resource list-types
```

Output:

> ```plain
> +---------------+-------------------------------------------+
> | resource_type | resource_controller_url                   |
> +---------------+-------------------------------------------+
> | generic       | http://127.0.0.1:8041/v1/resource/generic |
> +---------------+-------------------------------------------+
> ```

```bash
gnocchi resource-type create custom_resource_type
```

Output:

> ```
> +-------+----------------------+
> | Field | Value                |
> +-------+----------------------+
> | name  | custom_resource_type |
> +-------+----------------------+
> ```

```bash
echo "SELECT * FROM resource_type\x" | psql -U gnocchi gnocchi  
```

Output:

> ```plain
> Expanded display is on.
> -[ RECORD 1 ]-----------------------------------
> name       | generic
> tablename  | generic
> attributes | {}
> -[ RECORD 2 ]-----------------------------------
> name       | custom_resource_type
> tablename  | rt_f385f274c6c744048a012084926e6cae
> attributes | {}
> ```

```bash
gnocchi resource list-types
```

Output:

> ```plain
> +----------------------+--------------------------------------------------------+
> | resource_type        | resource_controller_url                                |
> +----------------------+--------------------------------------------------------+
> | generic              | http://127.0.0.1:8041/v1/resource/generic              |
> | custom_resource_type | http://127.0.0.1:8041/v1/resource/custom_resource_type |
> +----------------------+--------------------------------------------------------+
> ```


### Create a new resource

```bash
gnocchi resource create --attribute id:5a301761-f78b-46e2-8900-8b4f6fe6675a --attribute project_id:eba5c38f-c3dd-4d9c-9235-32d430471f94 -n temperature:high instance
```


Output:

> ```plain
> +-----------------------+---------------------------------------------------+
> | Field                 | Value                                             |
> +-----------------------+---------------------------------------------------+
> | created_by_project_id | c8d27445-48af-457c-8e0d-1de7103eae1f              |
> | created_by_user_id    | 99aae-4dc2-4fbc-b5b8-9688c470d9cc                 |
> | ended_at              | None                                              |
> | id                    | 5a301761-f78b-46e2-8900-8b4f6fe6675a              |
> | metrics               | temperature: 94a99b10-16f4-4a1c-b6cb-ea80956a828c |
> | original_resource_id  | 5a301761-f78b-46e2-8900-8b4f6fe6675a              |
> | project_id            | eba5c38f-c3dd-4d9c-9235-32d430471f94              |
> | revision_end          | None                                              |
> | revision_start        | 2016-05-12T23:53:38.291847+00:00                  |
> | started_at            | 2016-05-12T23:53:38.291765+00:00                  |
> | type                  | generic                                           |
> | user_id               | None                                              |
> +-----------------------+---------------------------------------------------+
> ```

```bash
echo "SELECT * FROM resource\x" | psql -U gnocchi gnocchi
echo "SELECT * FROM metric\x" | psql -U gnocchi gnocchi 
```

Output:

> ```plain
> Expanded display is on.
> -[ RECORD 1 ]---------+-------------------------------------
> created_by_user_id    | 99aae-4dc2-4fbc-b5b8-9688c470d9cc
> created_by_project_id | c8d27445-48af-457c-8e0d-1de7103eae1f
> started_at            | 2016-05-12 23:53:38.291765
> revision_start        | 2016-05-12 23:53:38.291847
> ended_at              | 
> user_id               | 
> project_id            | eba5c38f-c3dd-4d9c-9235-32d430471f94
> original_resource_id  | 5a301761-f78b-46e2-8900-8b4f6fe6675a
> id                    | 5a301761-f78b-46e2-8900-8b4f6fe6675a
> type                  | generic
> 
> Expanded display is on.
> -[ RECORD 1 ]---------+-------------------------------------
> id                    | 94a99b10-16f4-4a1c-b6cb-ea80956a828c
> archive_policy_name   | high
> created_by_user_id    | 99aae-4dc2-4fbc-b5b8-9688c470d9cc
> created_by_project_id | c8d27445-48af-457c-8e0d-1de7103eae1f
> resource_id           | 5a301761-f78b-46e2-8900-8b4f6fe6675a
> name                  | temperature
> status                | active
> ```


```bash
gnocchi resource create --attribute project_id:eba5c38f-c3dd-4d9c-9235-32d430471f94 -n temperature:high instance
```

Output:

> ```plain
> +-----------------------+---------------------------------------------------+
> | Field                 | Value                                             |
> +-----------------------+---------------------------------------------------+
> | created_by_project_id | c8d27445-48af-457c-8e0d-1de7103eae1f              |
> | created_by_user_id    | 99aae-4dc2-4fbc-b5b8-9688c470d9cc                 |
> | ended_at              | None                                              |
> | id                    | aeee00cb-8d3c-5d25-a55b-eb86709864a3              |
> | metrics               | temperature: 9f6d888c-3b7d-46ba-9bd0-5f92ec826784 |
> | original_resource_id  | instance                                          |
> | project_id            | eba5c38f-c3dd-4d9c-9235-32d430471f94              |
> | revision_end          | None                                              |
> | revision_start        | 2016-05-13T22:56:02.037365+00:00                  |
> | started_at            | 2016-05-13T22:56:02.037280+00:00                  |
> | type                  | generic                                           |
> | user_id               | None                                              |
> +-----------------------+---------------------------------------------------+
> ```

Note: If you don't provide a `--atribute id:<UUID>` then Gnocchi client 
    creates a UUID from the `'instance'` string and sends it to the API as
    the ID. So if you use the same ID on different calls, your string will be 
    converted to the same UUID.

See: <https://github.com/openstack/python-gnocchiclient/blob/fa7312ef23f48f8baf600428bfb788f02f10c74c/gnocchiclient/v1/resource.py#L77>


### List resources

```bash
gnocchi resource list
```

Output:

> ```plain
> +------------------------------+---------+------------------------------+---------+------------------------------+------------------------------+----------+--------------------------------+--------------+
> | id                           | type    | project_id                   | user_id | original_resource_id         | started_at                   | ended_at | revision_start                 | revision_end |
> +------------------------------+---------+------------------------------+---------+------------------------------+------------------------------+----------+--------------------------------+--------------+
> | 5a301761-f78b-               | generic | eba5c38f-c3dd-               | None    | 5a301761-f78b-               | 2016-05-13T21:26:46.680496+0 | None     | 2016-05-13T21:26:46.680575+00: | None         |
> | 46e2-8900-8b4f6fe6675a       |         | 4d9c-9235-32d430471f94       |         | 46e2-8900-8b4f6fe6675a       | 0:00                         |          | 00                             |              |
> | aeee00cb-8d3c-5d25-a55b-     | generic | eba5c38f-c3dd-               | None    | instance                     | 2016-05-13T22:56:02.037280+0 | None     | 2016-05-13T22:56:02.037365+00: | None         |
> | eb86709864a3                 |         | 4d9c-9235-32d430471f94       |         |                              | 0:00                         |          | 00                             |              |
> +------------------------------+---------+------------------------------+---------+------------------------------+------------------------------+----------+--------------------------------+--------------+
> ```


### List metrics

```bash
gnocchi metric list
```

Output:

> ```plain
> +--------------------------------------+---------------------+-------------+--------------------------------------+
> | id                                   | archive_policy/name | name        | resource_id                          |
> +--------------------------------------+---------------------+-------------+--------------------------------------+
> | 8cf29899-485e-434f-9408-07713c9156b4 | high                | temperature | 5a301761-f78b-46e2-8900-8b4f6fe6675a |
> | 9f6d888c-3b7d-46ba-9bd0-5f92ec826784 | high                | temperature | aeee00cb-8d3c-5d25-a55b-eb86709864a3 |
> +--------------------------------------+---------------------+-------------+--------------------------------------+
> ```


### Show a metric

Note: You can use the `original_resource_id` or the generated 'real' ID.
    Under the hood it generates the same REST request.

```bash
gnocchi resource show instance; gnocchi resource show aeee00cb-8d3c-5d25-a55b-eb86709864a3
```

Output:

> ```plain
> +-----------------------+---------------------------------------------------+
> | Field                 | Value                                             |
> +-----------------------+---------------------------------------------------+
> | created_by_project_id | c8d27445-48af-457c-8e0d-1de7103eae1f              |
> | created_by_user_id    | 99aae-4dc2-4fbc-b5b8-9688c470d9cc                 |
> | ended_at              | None                                              |
> | id                    | aeee00cb-8d3c-5d25-a55b-eb86709864a3              |
> | metrics               | temperature: 9f6d888c-3b7d-46ba-9bd0-5f92ec826784 |
> | original_resource_id  | instance                                          |
> | project_id            | eba5c38f-c3dd-4d9c-9235-32d430471f94              |
> | revision_end          | None                                              |
> | revision_start        | 2016-05-13T22:56:02.037365+00:00                  |
> | started_at            | 2016-05-13T22:56:02.037280+00:00                  |
> | type                  | generic                                           |
> | user_id               | None                                              |
> +-----------------------+---------------------------------------------------+
> +-----------------------+---------------------------------------------------+
> | Field                 | Value                                             |
> +-----------------------+---------------------------------------------------+
> | created_by_project_id | c8d27445-48af-457c-8e0d-1de7103eae1f              |
> | created_by_user_id    | 99aae-4dc2-4fbc-b5b8-9688c470d9cc                 |
> | ended_at              | None                                              |
> | id                    | aeee00cb-8d3c-5d25-a55b-eb86709864a3              |
> | metrics               | temperature: 9f6d888c-3b7d-46ba-9bd0-5f92ec826784 |
> | original_resource_id  | instance                                          |
> | project_id            | eba5c38f-c3dd-4d9c-9235-32d430471f94              |
> | revision_end          | None                                              |
> | revision_start        | 2016-05-13T22:56:02.037365+00:00                  |
> | started_at            | 2016-05-13T22:56:02.037280+00:00                  |
> | type                  | generic                                           |
> | user_id               | None                                              |
> +-----------------------+---------------------------------------------------+
> ```


### Push lots of measures to a metric

Use the `benchmark measures add` command. 10K measures might be a bit much. :)

```bash
gnocchi benchmark measures add --workers 10 --count 10000 --timestamp-start 2016-05-10T23:38:32+00:00  --timestamp-end 2016-05-13T23:38:32+00:00 --wait 8cf29899-485e-434f-9408-07713c9156b4
```

Output:

> ```plain
> +--------------------------------+---------------------+
> | Field                          | Value               |
> +--------------------------------+---------------------+
> | client workers                 | 10                  |
> | extra wait to process measures | 2.916589084 seconds |
> | measures per request           | 1                   |
> | measures push speed            | 9.54 push/s         |
> | push executed                  | 10000               |
> | push failures                  | 0                   |
> | push failures rate             | 0.00 %              |
> | push latency 95%'ile           | 1.3881878719        |
> | push latency 99%'ile           | 1.55702824551       |
> | push latency 99.9%'ile         | 1.93411220136       |
> | push latency max               | 2.152579248         |
> | push latency mean              | 1.04683287349       |
> | push latency median            | 1.0299056915        |
> | push latency min               | 0.510283199999      |
> | push runtime                   | 1048.18 seconds     |
> | push speed                     | 9.54 push/s         |
> +--------------------------------+---------------------+
> ```

Or by using batches of 10 (goes much faster):

```bash
gnocchi benchmark measures add --workers 10 --count 10000 --batch 10 --timestamp-start 2016-05-10T23:38:32+00:00  --timestamp-end 2016-05-13T23:38:32+00:00 --wait 8cf29899-485e-434f-9408-07713c9156b4
```

Output:

> ```plain
> +--------------------------------+---------------------+
> | Field                          | Value               |
> +--------------------------------+---------------------+
> | client workers                 | 10                  |
> | extra wait to process measures | 5.362727705 seconds |
> | measures per request           | 10                  |
> | measures push speed            | 100.57 push/s       |
> | push executed                  | 1000                |
> | push failures                  | 0                   |
> | push failures rate             | 0.00 %              |
> | push latency 95%'ile           | 1.30293057735       |
> | push latency 99%'ile           | 1.44964173241       |
> | push latency 99.9%'ile         | 1.60658611554       |
> | push latency max               | 1.609546691         |
> | push latency mean              | 0.989164812298      |
> | push latency median            | 0.986007109         |
> | push latency min               | 0.487609228001      |
> | push runtime                   | 99.44 seconds       |
> | push speed                     | 10.06 push/s        |
> +--------------------------------+---------------------+
> ```


### Get measures for a metric

```bash
gnocchi measures show 8cf29899-485e-434f-9408-07713c9156b4
```

Output:

> ```plain
> +---------------------------+-------------+----------------+
> | timestamp                 | granularity |          value |
> +---------------------------+-------------+----------------+
> | 2016-05-11T06:00:00+00:00 |      3600.0 |  299797007.712 |
> | 2016-05-11T07:00:00+00:00 |      3600.0 |  28931645.6944 |
> | 2016-05-11T08:00:00+00:00 |      3600.0 | -113101319.368 |
> | 2016-05-11T09:00:00+00:00 |      3600.0 |  304763864.549 |
> | 2016-05-11T10:00:00+00:00 |      3600.0 |  25061567.4514 |
> | 2016-05-11T11:00:00+00:00 |      3600.0 | -134661925.069 |
> | 2016-05-11T12:00:00+00:00 |      3600.0 | -178095964.653 |
> | 2016-05-11T13:00:00+00:00 |      3600.0 |  21855926.1329 |
> | 2016-05-11T14:00:00+00:00 |      3600.0 |  356511395.826 |
> | 2016-05-11T15:00:00+00:00 |      3600.0 |  28860546.5347 |
> ```

... etc.

> ```plain
> | 2016-05-14T04:03:32+00:00 |         1.0 |   1618440711.0 |
> | 2016-05-14T04:03:57+00:00 |         1.0 |   3603764972.0 |
> | 2016-05-14T04:04:22+00:00 |         1.0 |   2663127672.0 |
> | 2016-05-14T04:04:47+00:00 |         1.0 |   2748734737.0 |
> +---------------------------+-------------+----------------+
> ```


----

## CloudKitty

### Install & Configure

TODO

#### Run unit tests

```bash
cd ~/dev/spikes/gnocchi_cloudkitty/cloudkitty
workon cloudkitty
pip install -r test-requirements.txt
testr init
testr run
```


### Make CloudKitty play with Gnocchi

#### You need an 'instance' resource type

```bash
gnocchi resource-type create instance
gnocchi resource list-types
```

Output:

> ```plain
> +-------+----------+
> | Field | Value    |
> +-------+----------+
> | name  | instance |
> +-------+----------+
> +----------------------+--------------------------------------------------------+
> | resource_type        | resource_controller_url                                |
> +----------------------+--------------------------------------------------------+
> | generic              | http://127.0.0.1:8041/v1/resource/generic              |
> | instance             | http://127.0.0.1:8041/v1/resource/instance             |
> | custom_resource_type | http://127.0.0.1:8041/v1/resource/custom_resource_type |
> +----------------------+--------------------------------------------------------+
> ```


#### Create a new resource with this type 'instance'

```bash
gnocchi resource create --type instance --attribute project_id:eba5c38f-c3dd-4d9c-9235-32d430471f94 --attribute user_id:99aae-4dc2-4fbc-b5b8-9688c470d9cc -n cpu_cores:high instance01
gnocchi resource list -f yaml --type instance
gnocchi metric show --resource-id 4d1db66e-647e-5d23-95ff-3b66addbb9c1 -f yaml cpu_cores
```

Output:

> ```plain
> +-----------------------+-------------------------------------------------+
> | Field                 | Value                                           |
> +-----------------------+-------------------------------------------------+
> | created_by_project_id | c8d27445-48af-457c-8e0d-1de7103eae1f            |
> | created_by_user_id    | 99aae-4dc2-4fbc-b5b8-9688c470d9cc               |
> | ended_at              | None                                            |
> | id                    | 4d1db66e-647e-5d23-95ff-3b66addbb9c1            |
> | metrics               | cpu_cores: ce1394cb-7471-4a40-bc3f-001345e89556 |
> | original_resource_id  | instance01                                      |
> | project_id            | eba5c38f-c3dd-4d9c-9235-32d430471f94            |
> | revision_end          | None                                            |
> | revision_start        | 2016-05-18T04:31:33.937135+00:00                |
> | started_at            | 2016-05-18T04:31:33.937059+00:00                |
> | type                  | instance                                        |
> | user_id               | 99aae-4dc2-4fbc-b5b8-9688c470d9cc               |
> +-----------------------+-------------------------------------------------+
> 
> - ended_at: null
>   id: 4d1db66e-647e-5d23-95ff-3b66addbb9c1
>   original_resource_id: instance01
>   project_id: eba5c38f-c3dd-4d9c-9235-32d430471f94
>   revision_end: null
>   revision_start: '2016-05-18T04:31:33.937135+00:00'
>   started_at: '2016-05-18T04:31:33.937059+00:00'
>   type: instance
>   user_id: 99aae-4dc2-4fbc-b5b8-9688c470d9cc
> 
> archive_policy/aggregation_methods: std, count, 95pct, min, max, sum, median, mean
> archive_policy/back_window: 0
> archive_policy/definition: '- points: 3600, granularity: 0:00:01, timespan: 1:00:00
> 
>   - points: 10080, granularity: 0:01:00, timespan: 7 days, 0:00:00
> 
>   - points: 8760, granularity: 1:00:00, timespan: 365 days, 0:00:00'
> archive_policy/name: high
> created_by_project_id: c8d27445-48af-457c-8e0d-1de7103eae1f
> created_by_user_id: 99aae-4dc2-4fbc-b5b8-9688c470d9cc
> id: ce1394cb-7471-4a40-bc3f-001345e89556
> name: cpu_cores
> resource/created_by_project_id: c8d27445-48af-457c-8e0d-1de7103eae1f
> resource/created_by_user_id: 99aae-4dc2-4fbc-b5b8-9688c470d9cc
> resource/ended_at: null
> resource/id: 4d1db66e-647e-5d23-95ff-3b66addbb9c1
> resource/original_resource_id: instance01
> resource/project_id: eba5c38f-c3dd-4d9c-9235-32d430471f94
> resource/revision_end: null
> resource/revision_start: '2016-05-18T04:31:33.937135+00:00'
> resource/started_at: '2016-05-18T04:31:33.937059+00:00'
> resource/type: instance
> resource/user_id: 99aae-4dc2-4fbc-b5b8-9688c470d9cc
> ```


#### Add some measures for this metric

```bash
gnocchi benchmark measures add --workers 1 --count 24 --timestamp-start 2016-05-17T00:00:00+00:00  --timestamp-end 2016-05-18T00:00:00+00:00 --wait ce1394cb-7471-4a40-bc3f-001345e89556
```


#### Test CloudKitty requests

WIP: No resources returned by default CloudKitty request:

```bash
curl -H "Host: 127.0.0.1:8041" -H "x-project-id: c8d27445-48af-457c-8e0d-1de7103eae1f" -H "Accept: application/json, */*" -H "User-Agent: keystoneauth1/2.4.0 python-requests/2.9.1 CPython/2.7.11" -H "x-user-id: 99aae-4dc2-4fbc-b5b8-9688c470d9cc" -H "Content-Type: application/json" --data-binary '{"and": [{"or": [{"=": {"ended_at": null}}, {">=": {"ended_at": 1462104000}}]}, {"or": [{"=": {"ended_at": null}}, {"<=": {"ended_at": 1462107600}}]}, {"<=": {"started_at": 1462107600}}, {"<=": {"revision_start": 1462107600}}, {"=": {"id": "4d1db66e-647e-5d23-95ff-3b66addbb9c1"}}]}' --compressed "http://127.0.0.1:8041/v1/search/resource/instance?history=true&limit=1&sort=revision_start%3Adesc"
```

But when I modify it to this then it works:

```bash
curl -H "Host: 127.0.0.1:8041" -H "x-project-id: c8d27445-48af-457c-8e0d-1de7103eae1f" -H "Accept: application/json, */*" -H "User-Agent: keystoneauth1/2.4.0 python-requests/2.9.1 CPython/2.7.11" -H "x-user-id: 99aae-4dc2-4fbc-b5b8-9688c470d9cc" -H "Content-Type: application/json" --data-binary '{
	"and": [
    {
		"=": {
			"type": "instance"
		}
	}, {
		"=": {
			"project_id": "eba5c38f-c3dd-4d9c-9235-32d430471f94"
		}
	}]
}' --compressed http://127.0.0.1:8041/v1/search/resource/generic
```




----

## References

- <http://gnocchi.xyz/index.html>
- <http://gnocchi.xyz/gnocchiclient/>


## Troubleshooting

### 403 when checking Gnocchi status

```bash
curl http://127.0.0.1:8041/v1/status
```

Output:

> ```
> 403 Forbidden
> 
> Access was denied to this resource.
> ```

You probably forgot to add the `x-roles: admin` header.
