[![Known Vulnerabilities](https://snyk.io/test/github/cloudfoundry-community/cf-containers-broker/badge.svg)](https://snyk.io/test/github/cloudfoundry-community/cf-containers-broker)

# Containers Service Broker for Cloud Foundry

This is a generic `Containers` broker for the Cloud Foundry [v2 services API](http://docs.cloudfoundry.org/services/api.html).

This service broker allows users to provision services that runs inside a
[compatible container backend](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/README.md#prerequisites)
and bind applications to the service. The management tasks that the broker can perform are:

 * Provision a service container with random credentials and service arbitrary parameters
 * Bind a service container to an application:
    * Expose the credentials to access the provisioned service (see [CREDENTIALS.md](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/CREDENTIALS.md) for details)
    * Provide a syslog drain service for your application logs (see [SYSLOG_DRAIN.md](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/SYSLOG_DRAIN.md) for details)
 * Unbind a service container from an application
 * Unprovision a service container
 * Expose a service container management dashboard

More details can be found at this [Pivotal P.O.V Blog post](https://content.pivotal.io/blog/docker-service-broker-for-cloud-foundry).

## Disclaimer

This is not presently a production ready service broker. This is a work in progress. It is suitable for
experimentation and may not become supported in the future.

## Usage

### Prerequisites

This service broker does not include any container backend. Instead, it is meant to be deployed alongside any
compatible container backend, which it manages:

 * [Docker](https://www.docker.com/): Instructions to configure the service broker with a Docker backend can be found
  at [DOCKER.md](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/DOCKER.md).

### Configuration

Configure the application settings according to the instructions found at [SETTINGS.md](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/SETTINGS.md).

### Run

#### Standalone

Start the service broker:

```
bundle
bundle exec rackup
```

The service broker will listen by default at port 9292. View the catalog API at [http://localhost:9292/v2/catalog](http://localhost:9292v2/catalog). The basic auth username is `containers` and secret is `secret`.

#### As a Docker container

##### Build the image

This step is optional, you can use the already built Docker image located at the
[Docker Hub Registry](https://registry.hub.docker.com/u/frodenas/cf-containers-broker/).

If you want to create locally the image `frodenas/cf-containers-broker`
([Dockerfile](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/Dockerfile)) execute the following
command on a local cloned `cf-containers-broker` repository:

```
docker build -t frodenas/cf-containers-broker .
```

##### Run the image

To run the image and bind it to host port 80:

```
docker run -d --name cf-containers-broker \
       --publish 80:80 \
       --volume /var/run:/var/run \
       frodenas/cf-containers-broker
```

Some aspects of configuration can be overridden with environment variables. See `config/settings.yml` for documentation and environment variables.

```
docker run -d --name cf-containers-broker \
       --publish 80:80 \
       --volume /var/run:/var/run \
       -e BROKER_USERNAME=broker \
       -e BROKER_PASSWORD=password \
       -e EXTERNAL_HOST=localhost \
       frodenas/cf-containers-broker
```

If you want to override the entire configuration, then create a directory with the configuration files, and mount this directory into the container's `/config` directory:

```
mkdir -p /tmp/cf-containers-broker/config
cp config/settings.yml /tmp/cf-containers-broker/config
cp config/unicorn.conf.rb /tmp/cf-containers-broker/config
vi /tmp/cf-containers-broker/config/settings.yml
docker run -d --name cf-containers-broker \
       --publish 80:80 \
       --volume /var/run:/var/run \
       --volume /tmp/cf-containers-broker/config:/config \
       frodenas/cf-containers-broker
```

If you want to expose the application logs, create a host directory and mount the container's directory `/app/log`
into the previous host directory:

```
mkdir -p /tmp/cf-containers-broker/logs
docker run -d --name cf-containers-broker \
       --publish 80:80 \
       --volume /var/run:/var/run \
       --volume /tmp/cf-containers-broker/logs:/app/log \
       frodenas/cf-containers-broker
```


#### Using CF/BOSH

This service broker can be deployed alongside:

* [Docker CF-BOSH release](https://github.com/cloudfoundry-community/docker-boshrelease) if you plan to use Docker as backend.

### Enable the service broker at your Cloud Foundry environment

Add the service broker to Cloud Foundry as described by [the service broker documentation](http://docs.cloudfoundry.org/services/managing-service-brokers.html).

A quick way to register the service broker and to enable all service offerings is running:

```
cf create-service-broker docker-broker containers secret http://<BROKER IP ADDRESS>
while read p __; do
    cf enable-service-access "$p";
done < <(cf service-access | awk '/orgs/{y=1;next}y && NF' | sort | uniq)
```

### Bindings

The way that each service is configured determines how binding credentials are generated.

A service that exposes only a single port and has no other credentials configuration will include the minimal host and port in its credentials:

```json
{ "host": "10.11.12.13", "port": 61234, "ports": ["8080/tcp": 61234] }
```

In the example above, the container exposed an internal port `8080` and it was bound to port `61234` on the host machine `10.11.12.13`.

If a service exposes more than a single port, then you must specify the port you want to bind using the `credentials.uri.port` property,
otherwise the binding will not contain a port.

```json
{ "host": "10.11.12.13", "port": 61234, "ports": ["8080/tcp": 61234, "8081/tcp": 61235] }
```

In the example above, the container exposed internal ports `8080` and `8081`, and it was bound to port `61234` on the
host machine `10.11.12.13` because the `credentials.uri.port` property was set to `8080/tcp`.

For more details, see the [CREDENTIALS.md](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/CREDENTIALS.md) file.

#### Self-discovery of host port bindings

Optionally, each exposed host port for an instantiated container will passed into the container via environment variables, if you enable the `enable_host_port_envvar: true` setting.

If a Docker image exposes an internal port `5432`, then each instantiated container will be provided a `DOCKER_HOST_PORT_5432` environment variable containing the host's port allocation.

Implementation detail: In order to support this feature, provisioning new Docker containers requires two steps:

1. Instantiate a Docker container and allow Docker to allocate host ports.
2. Restart the Docker container with the additional `DOCKER_HOST_PORT_nnnn` environment variables.

If you wish to enable this feature, provide `enable_host_port_envvar: true` in `config/settings.yml`.

### Updating Containers

When new images become available or configuration of the plans change it can become desirable to restart the running containers to pick up the latest version of their image and/or update their configuration.

`bin/update_all_containers` will attempt to find all running containers managed by the broker and restart them with the latest configuration.

The mapping between running containers and configured plans is achieved by adding the labels `plan_id` and `instance_id` to the containers at create time. Containers that don't have these labels will be ignored by the update script.

If you are updating cf-containers-broker from an older version that didn't add the required labels you can force the broker to recreate the containers via `cf update-service <service-id>`. After this the labels will be available and the `bin/update_all_containers` script will be able to identify the containers for automatic updating.

In order for `cf update-service <service-id>` to work the service must declare `plan_updateable: true`.


### Tests

To run all specs:

```
bundle
bundle exec rake spec
```

Be aware that this project does not yet provide a full set of tests. Contributions are welcomed!

## Contributing

In the spirit of [free software](http://www.fsf.org/licensing/essays/free-sw.html), **everyone** is encouraged to help
improve this project.

Here are some ways *you* can contribute:

* by using alpha, beta, and prerelease versions
* by reporting bugs
* by suggesting new features
* by writing or editing documentation
* by writing specifications
* by writing code (**no patch is too small**: fix typos, add comments, clean up inconsistent whitespace)
* by refactoring code
* by closing [issues](https://github.com/cloudfoundry-community/cf-containers-broker/issues)
* by reviewing patches


### Submitting an Issue

We use the [GitHub issue tracker](https://github.com/cloudfoundry-community/cf-containers-broker/issues) to track bugs and
features. Before submitting a bug report or feature request, check to make sure it hasn't already been submitted. You
can indicate support for an existing issue by voting it up. When submitting a bug report, please include a
[Gist](http://gist.github.com/) that includes a stack trace and any details that may be necessary to reproduce the bug,
including your gem version, Ruby version, and operating system. Ideally, a bug report should include a pull request
with failing specs.

### Submitting a Pull Request

1. Fork the project.
2. Create a topic branch.
3. Implement your feature or bug fix.
4. Commit and push your changes.
5. Submit a pull request.

## Copyright

See [LICENSE](https://github.com/cloudfoundry-community/cf-containers-broker/blob/master/LICENSE) for details.
Copyright (c) 2014 [Pivotal Software, Inc](http://www.gopivotal.com/).
