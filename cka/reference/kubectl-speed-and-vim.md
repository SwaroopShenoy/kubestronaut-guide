# kubectl Speed and vim

Up: [CKA hub](../README.md) · Reference

## Aliases and completion

Set these at the start of the exam session:

```bash
alias k=kubectl
complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"
```

Using `$do` makes generating YAML quicker: `k create deploy web --image=nginx $do > web.yaml`.

## Imperative commands worth knowing cold

```bash
k run nginx --image=nginx --port=80 --labels="app=nginx"
k run nginx --image=busybox --command -- sleep 3600
k create deploy web --image=nginx --replicas=3
k expose deploy web --port=80 --target-port=8080
k create configmap cfg --from-literal=KEY=value
k create secret generic creds --from-literal=password=secret
k create sa builder
k create role r --verb=get,list --resource=pods
k create rolebinding rb --role=r --user=jane
k create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(200)'
k create cronjob backup --image=busybox --schedule="0 2 * * *" -- sh -c 'date'
k create namespace dev
```

## Quick edits

```bash
k edit deploy web
k set image deploy/web web=nginx:1.28
k set resources deploy web --requests=cpu=200m,memory=256Mi
k set env deploy web MODE=prod
k scale deploy web --replicas=5
k label pod <pod> tier=frontend
k annotate pod <pod> owner=team-a
```

## Output shortcuts

```bash
k get pods -o wide
k get pods -o jsonpath='{.items[*].metadata.name}{"\n"}'
k get pods --sort-by=.metadata.creationTimestamp
k get pods -l app=web --show-labels
k get pods -A --field-selector=status.phase!=Running
```

## vim for YAML

Add to `~/.vimrc`:

```vim
set number
set expandtab
set tabstop=2
set shiftwidth=2
```

Useful keys: `dd` delete line, `yy` copy, `p` paste, `:%s/old/new/g` replace, `>>` indent, `u` undo, `:wq` save and quit.

## Time habits

- Generate with `--dry-run=client -o yaml`, edit, apply. Faster than writing from memory.
- Prefer `kubectl edit` for a one-field change to an existing object.
- Use `k explain <kind>.<field>` instead of guessing field names.
