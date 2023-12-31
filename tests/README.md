<a href="https://elest.io">
  <img src="https://elest.io/images/elestio.svg" alt="elest.io" width="150" height="75">
</a>

[![Discord](https://img.shields.io/static/v1.svg?logo=discord&color=f78A38&labelColor=083468&logoColor=ffffff&style=for-the-badge&label=Discord&message=community)](https://discord.gg/4T4JGaMYrD "Get instant assistance and engage in live discussions with both the community and team through our chat feature.")
[![Elestio examples](https://img.shields.io/static/v1.svg?logo=github&color=f78A38&labelColor=083468&logoColor=ffffff&style=for-the-badge&label=github&message=open%20source)](https://github.com/elestio-examples "Access the source code for all our repositories by viewing them.")
[![Blog](https://img.shields.io/static/v1.svg?color=f78A38&labelColor=083468&logoColor=ffffff&style=for-the-badge&label=elest.io&message=Blog)](https://blog.elest.io "Latest news about elestio, open source software, and DevOps techniques.")

# Airflow, verified and packaged by Elestio

[Airflow](https://github.com/apache/airflow/) is a feature rich, multi-protocol messaging and streaming broker

<img src="https://github.com/elestio-examples/airflow/raw/main/airflow.png" alt="airflow" width="800">

Deploy a <a target="_blank" href="https://elest.io/open-source/airflow">fully managed airflow</a> on <a target="_blank" href="https://elest.io/">elest.io</a> if you want automated backups, reverse proxy with SSL termination, firewall, automated OS & Software updates, and a team of Linux experts and open source enthusiasts to ensure your services are always safe, and functional.

[![deploy](https://github.com/elestio-examples/airflow/raw/main/deploy-on-elestio.png)](https://dash.elest.io/deploy?source=cicd&social=dockerCompose&url=https://github.com/elestio-examples/airflow)

# Why use Elestio images?

- Elestio stays in sync with updates from the original source and quickly releases new versions of this image through our automated processes.
- Elestio images provide timely access to the most recent bug fixes and features.
- Our team performs quality control checks to ensure the products we release meet our high standards.

# Usage

## Git clone

You can deploy it easily with the following command:

    git clone https://github.com/elestio-examples/airflow.git

Copy the .env file from tests folder to the project directory

    cp ./tests/.env ./.env

Edit the .env file with your own values.

Create data folders with correct permissions

    mkdir -p ./dags;
    mkdir -p ./logs;
    mkdir -p ./config;
    mkdir -p ./plugins;
    mkdir -p ./postgres-db-volume;

    chmod 777 ./dags
    chmod 777 ./logs
    chmod 777 ./config
    chmod 777 ./plugins
    chmod 777 ./postgres-db-volume

Run the project with the following command

    docker-compose up -d

You can access the Web UI at: `http://your-domain:9999`

## Docker-compose

Here are some example snippets to help you get started creating a container.

    version: '3.8'
    x-airflow-common:
    &airflow-common
    # In order to add custom dependencies or upgrade provider packages you can use your extended image.
    # Comment the image line, place your Dockerfile in the directory where you placed the docker-compose.yaml
    # and uncomment the "build" line below, Then run `docker-compose build` to build the images.
    image: elestio4test/airflow:${SOFTWARE_VERSION_TAG}
    # build: .
    environment:
        &airflow-common-env
        AIRFLOW__CORE__EXECUTOR: CeleryExecutor
        AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
        # For backward compatibility, with Airflow <2.3
        AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
        AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
        AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
        AIRFLOW__CORE__FERNET_KEY: ''
        AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
        AIRFLOW__CORE__LOAD_EXAMPLES: 'true'
        AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session'
        # yamllint disable rule:line-length
        # Use simple http server on scheduler for health checks
        # See https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/logging-monitoring/check-health.html#scheduler-health-check-server
        # yamllint enable rule:line-length
        AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
        # WARNING: Use _PIP_ADDITIONAL_REQUIREMENTS option ONLY for a quick checks
        # for other purpose (development, test and especially production usage) build/extend Airflow image.
        _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
    volumes:
        - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
        - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
        - ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config
        - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
    user: "${AIRFLOW_UID:-50000}:0"
    depends_on:
        &airflow-common-depends-on
        redis:
        condition: service_healthy
        postgres:
        condition: service_healthy

    services:
    postgres:
        image: postgres:13
        environment:
        POSTGRES_USER: ${POSTGRES_USER:-airflow}
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-airflow}
        POSTGRES_DB: ${POSTGRES_DB:-airflow}
        volumes:
        - postgres-db-volume:/var/lib/postgresql/data
        healthcheck:
        test: ["CMD", "pg_isready", "-U", "airflow"]
        interval: 10s
        retries: 5
        start_period: 5s
        restart: always

    redis:
        image: redis:${REDIS_VERSION:-latest}
        expose:
        - 6379
        healthcheck:
        test: ["CMD", "redis-cli", "ping"]
        interval: 10s
        timeout: 30s
        retries: 50
        start_period: 30s
        restart: always

    airflow-webserver:
        <<: *airflow-common
        command: webserver
        ports:
        - "172.17.0.1:9999:8080"
        healthcheck:
        test: ["CMD", "curl", "--fail", "http://172.17.0.1:9999/health"]
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s
        restart: always
        depends_on:
        <<: *airflow-common-depends-on
        airflow-init:
            condition: service_completed_successfully

    airflow-scheduler:
        <<: *airflow-common
        command: scheduler
        healthcheck:
        test: ["CMD", "curl", "--fail", "http://172.17.0.1:8974/health"]
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s
        restart: always
        depends_on:
        <<: *airflow-common-depends-on
        airflow-init:
            condition: service_completed_successfully

    airflow-worker:
        <<: *airflow-common
        command: celery worker
        healthcheck:
        test:
            - "CMD-SHELL"
            - 'celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s
        environment:
        <<: *airflow-common-env
        # Required to handle warm shutdown of the celery workers properly
        # See https://airflow.apache.org/docs/docker-stack/entrypoint.html#signal-propagation
        DUMB_INIT_SETSID: "0"
        restart: always
        depends_on:
        <<: *airflow-common-depends-on
        airflow-init:
            condition: service_completed_successfully

    airflow-triggerer:
        <<: *airflow-common
        command: triggerer
        healthcheck:
        test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s
        restart: always
        depends_on:
        <<: *airflow-common-depends-on
        airflow-init:
            condition: service_completed_successfully

    airflow-init:
        <<: *airflow-common
        entrypoint: /bin/bash
        # yamllint disable rule:line-length
        command:
        - -c
        - |
            function ver() {
            printf "%04d%04d%04d%04d" $${1//./ }
            }
            airflow_version=$$(AIRFLOW__LOGGING__LOGGING_LEVEL=INFO && gosu airflow airflow version)
            airflow_version_comparable=$$(ver $${airflow_version})
            min_airflow_version=2.2.0
            min_airflow_version_comparable=$$(ver $${min_airflow_version})
            if (( airflow_version_comparable < min_airflow_version_comparable )); then
            echo
            echo -e "\033[1;31mERROR!!!: Too old Airflow version $${airflow_version}!\e[0m"
            echo "The minimum Airflow version supported: $${min_airflow_version}. Only use this or higher!"
            echo
            exit 1
            fi
            if [[ -z "50000" ]]; then
            echo
            echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
            echo "If you are on Linux, you SHOULD follow the instructions below to set "
            echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
            echo "For other operating systems you can get rid of the warning with manually created .env file:"
            echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
            echo
            fi
            one_meg=1048576
            mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
            cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
            disk_available=$$(df / | tail -1 | awk '{print $$4}')
            warning_resources="false"
            if (( mem_available < 4000 )) ; then
            echo
            echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
            echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
            echo
            warning_resources="true"
            fi
            if (( cpus_available < 2 )); then
            echo
            echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
            echo "At least 2 CPUs recommended. You have $${cpus_available}"
            echo
            warning_resources="true"
            fi
            if (( disk_available < one_meg * 10 )); then
            echo
            echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
            echo "At least 10 GBs recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
            echo
            warning_resources="true"
            fi
            if [[ $${warning_resources} == "true" ]]; then
            echo
            echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
            echo "Please follow the instructions to increase amount of resources available:"
            echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
            echo
            fi
            mkdir -p /sources/logs /sources/dags /sources/plugins
            chown -R "50000:0" /sources/{logs,dags,plugins}
            exec /entrypoint airflow version
        # yamllint enable rule:line-length
        environment:
        <<: *airflow-common-env
        _AIRFLOW_DB_UPGRADE: 'true'
        _AIRFLOW_WWW_USER_CREATE: 'true'
        _AIRFLOW_WWW_USER_USERNAME: ${ADMIN_EMAIL:-airflow}
        _AIRFLOW_WWW_USER_PASSWORD: ${ADMIN_PASSWORD:-airflow}
        _PIP_ADDITIONAL_REQUIREMENTS: ''
        user: "0:0"
        volumes:
        - ${AIRFLOW_PROJ_DIR:-.}:/sources

    airflow-cli:
        <<: *airflow-common
        profiles:
        - debug
        environment:
        <<: *airflow-common-env
        CONNECTION_CHECK_MAX_COUNT: "0"
        # Workaround for entrypoint issue. See: https://github.com/apache/airflow/issues/16252
        command:
        - bash
        - -c
        - airflow

    # You can enable flower by adding "--profile flower" option e.g. docker-compose --profile flower up
    # or by explicitly targeted on the command line e.g. docker-compose up flower.
    # See: https://docs.docker.com/compose/profiles/
    flower:
        <<: *airflow-common
        command: celery flower
        profiles:
        - flower
        ports:
        - "172.17.0.1:5555:5555"
        healthcheck:
        test: ["CMD", "curl", "--fail", "http://172.17.0.1:5555/"]
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s
        restart: always
        depends_on:
        <<: *airflow-common-depends-on
        airflow-init:
            condition: service_completed_successfully

    volumes:
    postgres-db-volume:

### Environment variables

|       Variable       | Value (example)  |
| :------------------: | :--------------: |
|     ADMIN_EMAIL      | admin@gmail.com  |
|    ADMIN_PASSWORD    |  your-password   |
| SOFTWARE_VERSION_TAG |      latest      |
|    REDIS_VERSION     |      latest      |
|    POSTGRES_USER     |    your-user     |
|  POSTGRES_PASSWORD   | your-db-password |
|     POSTGRES_DB      |   your-db-name   |

# Maintenance

## Logging

The Elestio Airflow Docker image sends the container logs to stdout. To view the logs, you can use the following command:

    docker-compose logs -f

To stop the stack you can use the following command:

    docker-compose down

## Backup and Restore with Docker Compose

To make backup and restore operations easier, we are using folder volume mounts. You can simply stop your stack with docker-compose down, then backup all the files and subfolders in the folder near the docker-compose.yml file.

Creating a ZIP Archive
For example, if you want to create a ZIP archive, navigate to the folder where you have your docker-compose.yml file and use this command:

    zip -r myarchive.zip .

Restoring from ZIP Archive
To restore from a ZIP archive, unzip the archive into the original folder using the following command:

    unzip myarchive.zip -d /path/to/original/folder

Starting Your Stack
Once your backup is complete, you can start your stack again with the following command:

    docker-compose up -d

That's it! With these simple steps, you can easily backup and restore your data volumes using Docker Compose.

# Links

- <a target="_blank" href="https://github.com/apache/airflow/">Airflow Github repository</a>

- <a target="_blank" href="https://airflow.apache.org/docs/">Airflow documentation</a>

- <a target="_blank" href="https://github.com/elestio-examples/airflow">Elestio/Airflow Github repository</a>
