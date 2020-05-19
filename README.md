# InSpec for Container Scanning

This is a repository to showcase InSpec scanning of containers.

## Prerequisites

1. Chef Workstation installed (for the `inspec` cli)
2. Docker installed on your workstation

## Building the container image

The included `Dockerfile` will create a container image with PostgreSQL installed. To build this:

```
> docker build -t eg_postgresql .
```

## Running InSpec tests

The `postgres-tests` directory contains a simple InSpec profile to test our container. It depends on the DevSec Postgres Baseline (https://github.com/dev-sec/postgres-baseline), and includes a selection of controls.

First you need to run your container:

```
> docker run --rm -P --name pg_test -p 5432:5432 eg_postgresql
```

Next, open a new terminal and use `docker ps` to find your container ID:

```
> docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
b2d0fbbbf6b1        eg_postgresql       "/usr/lib/postgresql…"   6 minutes ago       Up 6 minutes        0.0.0.0:5432->5432/tcp   pg_test
```

Finally, execute InSpec tests:

```
> inspec exec postgres-tests -t docker://<container_id> --input-file inputs.yml
```

## Oh no, one of my tests failed!

You should see output similar to this:

```
Profile: InSpec Profile (postgres-tests)
Version: 0.1.0
Target:  docker://b2d0fbbbf6b17aca547ec84ba5e7c623f1923dd807f46b014834681ba7112e28

     No tests executed.

Profile: DevSec PostgreSQL Baseline (postgres-baseline)
Version: 2.0.4
Target:  docker://b2d0fbbbf6b17aca547ec84ba5e7c623f1923dd807f46b014834681ba7112e28

  ✔  postgres-02: Use stable postgresql version
     ✔  Command: `psql -V` stdout is expected to match /^psql\s\(PostgreSQL\)\s(9\.[3-6]|10\.5).*/
     ✔  Command: `psql -V` stdout is expected not to match /RC/
     ✔  Command: `psql -V` stdout is expected not to match /DEVEL/
     ✔  Command: `psql -V` stdout is expected not to match /BETA/
  ✔  postgres-03: Run one postgresql instance per operating system
     ✔  Processes postgres entries.length is expected to eq 1
  ✔  postgres-04: Only "c" and "internal" should be used as non-trusted procedural languages
     ✔  PostgreSQL query: SELECT count (*) FROM pg_language WHERE lanpltrusted = 'f' AND lanname!='internal' AND lanname!='c'; output is expected to eq "0"
  ×  postgres-05: Set a password for each user
     ×  PostgreSQL query: SELECT count(*) FROM pg_shadow WHERE passwd IS NULL; output is expected to eq "0"

     expected: "0"
          got: "1"

     (compared using ==)

  ✔  postgres-09: The PostgreSQL "data_directory" should be assigned exclusively to the database account (such as "postgres").
     ✔  Command: `find /var/lib/postgresql/9.3/main -user docker -group docker -perm /go=rwx` stdout is expected to eq ""


Profile Summary: 4 successful controls, 1 control failure, 0 controls skipped
Test Summary: 7 successful, 1 failure, 0 skipped
```

Don't fear, we can fix the failing test. In the `Dockerfile`, uncomment lines 35-36:

```
RUN    /etc/init.d/postgresql start &&\
     psql --command "ALTER USER postgres PASSWORD 'postgres';"
```

Rebuild your container image, and run the latest version. The tests should now pass when you execute again.

# What else can I do?

## Scanning a Docker Host to CIS benchmarks

You will need a running Chef Automate instance for this. First, log in with the `inspec` cli:

```
> inspec compliance login https://<AUTOMATE_URL>.com --insecure --user=<USERNAME> --token=<ADMIN_TOKEN> --ent=default
```

On the Automate dashboard, locate and install the `CIS Docker Benchmark Profile`

Rename the `reporter-example.json` file to `reporter.json` and enter your Chef Automate credentials.

Finally, run the scan:

```
> inspec exec compliance://<USERNAME>/cis-docker-benchmark --config reporter.json
```

View the Compliance tab on Chef Automate to see the scan results

## Using the `docker_container` resource

Check our container configuration:

```
describe docker_container('pg_test') do
  it { should exist }
  it { should be_running }
  its('repo') { should eq 'eg_postgresql' }
  its('ports') { should eq "0.0.0.0:5432->5432/tcp" }
end
```

## Using the `docker_image` resource

Checking a specific version of an image:

```
describe docker_image('alpine:latest') do
  it { should exist }
  its('id') { should eq 'sha256:4a415e3663882fbc554ee830889c68a33b3585503892cc718a4698e91ef2a526' }
  its('image') { should eq 'alpine:latest' }
  its('repo') { should eq 'alpine' }
  its('tag') { should eq 'latest' }
end
```

Checking that old, unsupported image doesn't exist:

```
describe docker_image('u12:latest') do
  it { should_not exist }
end
```

## Using the `docker` resource for advanced use cases

Ensure no running containers use old image:

```
describe docker.containers do
  its('images') { should_not include 'u12:latest' }
end
```

Check if `ADD` instruction was used in any images:

```
docker.images.ids.each do |id|
  describe command("docker history #{id}| grep 'ADD'") do
    its('stdout') { should eq '' }
  end
end
```