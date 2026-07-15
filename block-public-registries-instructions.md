# Block Public Registries — Air-Gap Simulation

Deploys a privileged DaemonSet that blocks `cp.icr.io` and `icr.io` by adding `/etc/hosts` entries on all nodes (masters + workers). No MCP rollout, no node reboots. Works on Fyre and TechZone clusters. Cleanup is automatic on removal.

## Deploy

```bash
oc apply -f - <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: block-public-registries
  namespace: kube-system
  labels:
    app: block-public-registries
spec:
  selector:
    matchLabels:
      app: block-public-registries
  template:
    metadata:
      labels:
        app: block-public-registries
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
      - operator: Exists
      containers:
      - name: blocker
        image: registry.access.redhat.com/ubi9/ubi
        command: ["/bin/bash", "-c"]
        args:
        - |
          MARKER="# BLOCK-REGISTRIES"

          block_registries() {
            if ! nsenter -t 1 -m -- grep -q "$MARKER" /etc/hosts; then
              nsenter -t 1 -m -- bash -c "echo -e '\n$MARKER\n127.0.0.9 cp.icr.io\n127.0.0.10 icr.io\n$MARKER-END' >> /etc/hosts"
              echo "$(hostname): registries blocked in /etc/hosts"
            else
              echo "$(hostname): already blocked"
            fi
          }

          cleanup() {
            echo "$(hostname): removing registry blocks from /etc/hosts..."
            nsenter -t 1 -m -- sed -i "/$MARKER/,/$MARKER-END/d" /etc/hosts
            echo "$(hostname): cleanup done"
            exit 0
          }

          trap cleanup SIGTERM SIGINT

          block_registries

          while true; do sleep 3600 & wait $!; done
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: 10m
            memory: 16Mi
          limits:
            cpu: 50m
            memory: 64Mi
      terminationGracePeriodSeconds: 30
EOF
```

## Verify Pods Running

```bash
oc get pods -n kube-system -l app=block-public-registries -o wide
```


## Remove

```bash
oc delete ds block-public-registries -n kube-system
```

`/etc/hosts` entries are cleaned up automatically via the SIGTERM trap.

## Verify Cleanup

```bash
oc debug node/<worker> -- chroot /host grep BLOCK /etc/hosts
```

Expected: no output.

## Modify Blocked Registries

Edit the `block_registries()` function in the deploy command — add/remove lines following the `127.0.0.x hostname` pattern between the markers. Then:

```bash
oc delete ds block-public-registries -n kube-system
# wait for cleanup, then re-apply the updated deploy command
```

## Notes

- MCO does not track `/etc/hosts` — no drift or degraded state.
- Confirmed safe: `oc get mc -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.config.storage.files[*]}  {.path}{"\n"}{end}{end}' | grep hosts` returns empty.
- Uses `nsenter -t 1 -m` to write to host filesystem from the privileged pod.
- Covers all nodes via `tolerations: [{operator: Exists}]`.
