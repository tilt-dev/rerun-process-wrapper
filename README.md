# Restarting Your Process Via `live_update`

Tilt's Live Update functionality allows you to copy files to, and run commands on, running containers. Some apps or invocations thereof (e.g. Javascript apps run via `nodemon`, or Flask apps run in debug mode) detect and incorporate code changes without needing to restart. For other apps, though, you’ll need to restart them for changes to take effect.

This set of scripts is one way to restart your process as part of a Live Update.

For more information on restarting processes during Live Update, [see the docs](https://docs.tilt.dev/live_update_reference.html#restarting-your-process)

## How to Use These Scripts

For an example, see the [the onewatch integration test](https://github.com/windmilleng/tilt/tree/master/integration/onewatch).

Copy `start.sh` and `restart.sh` to your container working dir.

If your container entrypoint *was* `path/to/binary [arg1] [arg2]...`, change it to:
```
./start.sh path/to/binary [arg1] [arg2]...
```

(The `entrypoint` parameter on `docker_build`/`custom_build` is a lightweight way to do this.)

To restart the container as part of your Live Update, use a `run` step that calls the restart script: `run('./restart.sh')`

So, for example:

```python
docker_build('gcr.io/windmill-test-containers/integration/onewatch',
    '.',
    entrypoint=['./start.sh', '/go/bin/onewatch'],
    live_update=[
        sync('.', '/go/src/github.com/windmilleng/tilt/integration/onewatch'),
        run('go install github.com/windmilleng/tilt/integration/onewatch'),
        run('./restart.sh'),
    ])
```

This live update will cause the `go install` to be run in the container every time anything in the `.` path locally changes. After the `go install` is run, `./restart.sh` will be run. This will kill the original entrypoint, and restart it, effectively simulating the `container_restart()` functionality on Docker.

## Non-Supported Cases

### Containers Without Shell
At the risk of stating the obvious: your container needs to have shell available to run these shell scripts.

These scripts look for a shell at `/bin/sh`; if your shell lives elsewhere, you have to explicitly invoke the scripts with that shell, e.g.:
```python
docker_build(...,
    entrypoint=['/busybox/sh', './start.sh', 'path/to/binary'],
    ...
)
```

### Subprocesses (Not Necessarily Supported)
These scripts will only work as well as signal handling between your parent process and its subprocesses; use with care.

Specifically, `restart.sh` only knows how to kill the top-level process invoked by `start.sh`; it's up to that process to clean up its subprocesses on receiving `SIGTERM`, or else you'll be left with orphans.
