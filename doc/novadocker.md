Nova Docker Support
===================

If you want to bootstrap a cloud with nova-docker installed you can do the following:

Environment
-----------

Add the following to `envs/example/group_vars/compute` to to use the nova docker driver only on compute nodes, otherwise add it to `envs/example/defaults.yml` for all nodes.

```
docker:
  enabled: true

novadocker:
  enabled: true
```

Then use `ursula` to boot your openstack cluster just as you would normally.

Images
------

Adding docker images is currently something only an Administrator should do.   It's quite easy (the glance name should be the same as the docker image name):

```
$ docker pull busybox && \
    docker save busybox | \
    glance image-create --is-public=True --container-format=docker \
    --disk-format=raw --name busybox
```

Then you should be able to boot a docker container like this:

```
$ nova boot --image=busybox --flavor=1 --nic net-id=<net-uuid> my_first_docker
```

You can check that it is actually running with

```
$ docker ps
```
