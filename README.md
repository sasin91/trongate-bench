# trongate-bench

Hello-world and session benchmarks for [Trongate](https://trongate.io) v2 apps on Kubernetes.
Extracted from the `bench/` kit in [trongate.cloud](https://github.com/sasin91/trongate.cloud).

It measures a stock Trongate v2 app (trongate-framework `cc968cf`) on the runtime trongate.cloud
gives customer apps: FrankenPHP (PHP 8.4.26) with Railpack's php.ini and Caddyfile (Railpack
v0.40.1), OPcache on. It covers:

- requests a second and latency against CPU limit, for Trongate and for a raw PHP baseline;
- the cost of Caddy's `DEBUG` log level against `INFO`;
- new visitors (no cookie, as `wrk` and `ab` send by default) against returning visitors (1000
  rotating session cookies);
- session handling: Trongate's `session_start()` on every request against lazy sessions, with
  files, Valkey and Memcached as the session store.

## Lazy sessions

`app/ignition.lazy.php` is the proposed trongate-framework change. It replaces
`engine/ignition.php` and starts a session only when the visitor already has a session cookie, or
when the request writes to `$_SESSION` (the response is buffered and a shutdown function starts
the session). Build with `--build-arg SESSION_START=lazy` to use it; `stock` keeps Trongate's own
`ignition.php`.

## Files

- `Dockerfile` builds stock Trongate v2 plus a `bench` module (`app/Bench.php`): `/bench` prints
  Hello World through Trongate's full bootstrap, `/raw.php` is the plain-PHP baseline.
  `PHP_SESSION_HANDLER`/`PHP_SESSION_PATH` pick files, Valkey (redis) or Memcached at runtime.
- `session-tests.sh` checks session behaviour (cookies, flashdata across a redirect, CSRF, late
  writes, no leaked locks) against an app on :8080.
- `users.lua` makes wrk act as 1000 returning visitors, each with their own session cookie.
- `push-images.sh` builds and pushes the app and wrk images; `cluster-run.sh` runs the whole set
  in a throwaway `bench` namespace (own Valkey and Memcached, wrk on a different node than the
  app) and deletes it afterwards. It refuses to run unless the kubectl context is
  `trongate-cloud` (override with `BENCH_CONTEXT`). Results go to `results/` (git-ignored).

## Run it locally

```sh
docker build -f Dockerfile --build-arg SESSION_START=lazy -t trongate-bench:lazy .
docker run --rm -p 8080:8080 trongate-bench:lazy
./session-tests.sh
```

## Run it on a cluster

```sh
./push-images.sh            # REGISTRY=... to push somewhere else
KUBECONFIG=~/.config/trongate-cloud/kubeconfig ./cluster-run.sh
```

trongate-bench is an independent project and is not affiliated with or endorsed by the Trongate
team.
