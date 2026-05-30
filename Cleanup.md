# Cleanup — Nebius DDP Training Homework

## Verify resources before deleting

```bash
nebius mk8s cluster list
nebius mk8s node-group list
nebius container-registry registry list
sky status
```

## Cleanup script

Run in order — dependencies must be deleted before parent resources:

```bash
# 1. Tear down the SkyPilot cluster
#    This terminates the Kubernetes pods that SkyPilot launched for the training job.
#    The underlying GPU VMs (node group) keep running and billing continues —
#    this step only stops the job, not the machines.
sky down ariel-ddp

# 2. Delete the mk8s node group
#    THIS IS WHERE GPU BILLING STOPS.
#    The node group owns the actual H100 GPU virtual machines.
#    Deleting it terminates all VMs in the group and stops per-second GPU charges immediately.
#    Must be deleted before the cluster (it is a child resource).
nebius mk8s node-group delete --id mk8snodegroup-e00wj78a8esqpqymnp

# 3. Delete the mk8s cluster
#    The cluster itself (control plane) has a small management fee.
#    Billing for the cluster stops once it is fully deleted (~1–2 min after this command).
#    Can only be deleted after all node groups are removed.
nebius mk8s cluster delete --id mk8scluster-e00qn3h3n6rqax9gr2

# 4. Delete the container registry
#    The registry charges for stored image data (GB/month).
#    Deleting it removes all images inside and stops storage billing immediately.
#    Safe to delete after the cluster is gone — nodes no longer need to pull from it.
nebius container-registry registry delete --id e00xbrt95y1x46pwr3
```

> **Warning:** These actions are irreversible. GPU billing stops when the node group is deleted.
